# Sistema Distribuído de Monitoramento com Drones

Projeto da disciplina de Redes de Computadores II — implementação de um sistema distribuído de despacho de drones para atendimento de ocorrências, com exclusão mútua distribuída via algoritmo de Ricart-Agrawala, fila de prioridade compartilhada entre múltiplos brokers e tolerância a falhas de brokers e drones.

---

## Visão Geral

O sistema simula um cenário de segurança pública em que sensores detectam ocorrências em diferentes setores e drones são despachados para atendê-las com base em prioridade. A solução é composta por quatro tipos de módulos independentes que se comunicam via TCP:

- **Broker** — coordena o despacho de drones, mantém a fila de requisições compartilhada e se comunica com os demais brokers para garantir consistência distribuída
- **Drone** — conecta-se a um broker disponível, aguarda missões e reporta conclusão ou queda
- **Sensor** — detecta ocorrências aleatórias e as envia ao broker do seu setor automaticamente
- **Cliente** — interface manual que substitui o sensor, permitindo ao operador enviar ocorrências escolhendo o tipo e setor

Cada setor (`setor_a`, `setor_b`, `setor_c`) tem seu próprio broker, drone e sensor. Apesar de independentes, todos compartilham uma fila de requisições distribuída, replicada e consistente entre os três brokers.

---

## Arquitetura

```
Sensor A ──► Broker A ◄──── TCP (Ricart-Agrawala) ────► Broker B ◄── Sensor B
                │                                             │
             Drone A                                       Drone B
                │                                             │
                └──────────────── Fila Compartilhada ─────────┘
                              (replicada via broadcast)
                                         │
                                      Broker C ◄── Sensor C
                                         │
                                      Drone C

Cliente (manual) ──► qualquer Broker (via porta 7xxx)
```

### Portas utilizadas

| Módulo | setor_a | setor_b | setor_c |
|--------|---------|---------|---------|
| Broker ↔ Broker (Ricart-Agrawala) | 5001 | 5002 | 5003 |
| Drone → Broker | 6001 | 6002 | 6003 |
| Sensor / Cliente → Broker | 7001 | 7002 | 7003 |

---

## Algoritmo de Ricart-Agrawala

A exclusão mútua distribuída é implementada pelo algoritmo de **Ricart-Agrawala**, garantindo que apenas um broker por vez modifique a fila compartilhada. Sem essa garantia, dois brokers poderiam despachar o mesmo drone simultaneamente para ocorrências diferentes, causando inconsistência.

### Funcionamento

1. Quando um broker quer acessar a seção crítica (adicionar ou remover da fila, despachar um drone), ele incrementa seu clock de Lamport e envia `REQUEST` com o timestamp para todos os peers
2. Cada peer responde com `REPLY` imediatamente se estiver `RELEASED`, ou adia a resposta se estiver `HELD` (na seção crítica) ou `WANTED` com timestamp menor (maior prioridade)
3. O broker que receber `REPLY` de todos os peers entra na seção crítica (`HELD`)
4. Ao sair, envia os `REPLY`s que foram adiados, liberando os peers que estavam esperando

### Relógio de Lamport

O timestamp lógico é mantido por um relógio de Lamport:
- Incrementado a cada evento local (`_incrementarClock`)
- Atualizado ao receber mensagens com `max(local, recebido) + 1` (`_atualizarClock`)

Isso garante ordenação causal dos eventos entre os três brokers, mesmo sem relógio físico sincronizado.

### O que o Ricart-Agrawala garante neste sistema

- **Exclusão mútua** — só um broker por vez opera sobre a fila
- **Ausência de deadlock** — o desempate por timestamp e ID do broker garante que sempre um dos brokers avança
- **Ausência de starvation** — todos os brokers eventualmente recebem permissão, pois as respostas adiadas são sempre enviadas na saída da seção crítica

---

## Fila de Prioridade Compartilhada

A fila é replicada em todos os brokers e mantida consistente via broadcast. Cada requisição possui um identificador único (`req_id`) no formato `setor_id:timestamp:uuid`, que permite:

- Merge idempotente durante sincronização (sem duplicatas)
- Remoção exata por ID ao atender uma requisição
- Desempate determinístico na ordenação

### Critérios de ordenação

| Critério | Direção | Descrição |
|----------|---------|-----------|
| Prioridade | Decrescente | `3 = ataque_concreto` > `2 = possivel_ataque` > `1 = averiguacao` |
| Timestamp de Lamport | Crescente | Quem chegou primeiro dentro da mesma prioridade |
| req_id | Crescente | Desempate final determinístico |

### Fluxo de uma requisição

```
Sensor ou Cliente detecta/envia ocorrência
        ↓
Broker recebe via TCP (porta 7xxx)
        ↓
Broker pede permissão ao Ricart-Agrawala
        ↓
[SEÇÃO CRÍTICA]
  Há drone livre e fila vazia?
  ├── SIM → despacha drone diretamente
  └── NÃO → insere na fila + broadcast para peers
[FIM DA SEÇÃO CRÍTICA]
        ↓
Drone conclui missão → broker processa próximo da fila
```

---

## Tolerância a Falhas

### Broker cai e volta

Quando um broker reinicia, ele não começa com fila vazia — executa uma **sincronização inicial tripla** com intervalo de 3 segundos para garantir que captura até as requisições que chegaram durante o próprio processo de sync:

```
sync 1/3 → pede fila para peers ativos
           (aguarda 3s — novas requisições podem ter chegado)
sync 2/3 → pede fila novamente para capturar o que chegou
           (aguarda 3s)
sync 3/3 → última rodada de nivelamento
```

O merge em cada sync é feito por `req_id` — requisições já presentes localmente não são duplicadas. Após os 9 segundos iniciais, o broker está nivelado com os peers e os broadcasts normais mantêm a consistência em tempo real.

**Importante:** as requisições **não são removidas da fila** quando um broker cai. Os brokers que ficaram ativos continuam com a fila íntegra e o broker que voltar a recebe completa via sync.

### Drone cai durante missão

Quando um drone desconecta enquanto está em missão ativa, o broker:

1. Detecta a queda via `recv()` retornando vazio no `_loopDrone`
2. Recupera a requisição salva em `ocorrencia_atual` (campo preenchido no momento do despacho)
3. Reinsere a requisição na fila respeitando a prioridade original
4. Faz broadcast `FILA_REQUISICAO` para que todos os outros brokers também reinsiram a requisição
5. Exibe o banner de recuperação no terminal

Se o drone desconectar sem missão ativa (apenas idle), nenhuma requisição é perdida e o sistema continua normalmente.

### Drone reconecta após queda de broker

O drone tenta se conectar a qualquer broker disponível na lista. Se o broker ao qual estava conectado cair, o drone automaticamente tenta o próximo da lista até encontrar um disponível. Ao reconectar, envia `CADASTRO` e o novo broker registra o drone e faz broadcast para os peers.

### Falha na comunicação entre brokers

Quando um broker tenta enviar mensagem a um peer e a conexão falha (timeout de 2 segundos), o envio é ignorado silenciosamente. O peer offline simplesmente não recebe o broadcast. Quando esse peer voltar, o sync inicial vai nivelar a fila.

---

## Protocolo de Mensagens

Toda comunicação entre módulos é feita via JSON sobre TCP.

| Tipo | Entre | Descrição |
|------|-------|-----------|
| `REQUEST` | Broker → Broker | Solicita acesso à seção crítica (Ricart-Agrawala) |
| `REPLY` | Broker → Broker | Concede acesso ao solicitante |
| `DRONE_UPDATE` | Broker → Broker | Notifica mudança de estado de um drone (livre/ocupado) |
| `FILA_REQUISICAO` | Broker → Broker | Broadcast de nova requisição adicionada à fila |
| `FILA_REMOVIDA` | Broker → Broker | Broadcast de requisição atendida (remove por req_id) |
| `FILA_SYNC_REQUEST` | Broker → Broker | Broker recém-iniciado pede a fila atual |
| `FILA_SYNC_RESPONSE` | Broker → Broker | Resposta com a fila completa para merge |
| `EXECUTE` | Broker → Broker | Pede a outro broker que despache um drone remoto |
| `BROKER_OFFLINE` | Broker → Broker | Avisa peers que um broker caiu (limpa fila de requisições do broker caído) |
| `CADASTRO` | Drone → Broker | Drone se registra ao conectar |
| `DISPATCH` | Broker → Drone | Envia missão ao drone |
| `CONCLUIDO` | Drone → Broker | Drone informa conclusão da missão |

---

## Como Rodar

### Pré-requisitos

- Python 3.10+
- Sem dependências externas (apenas biblioteca padrão)

### Localmente (uma máquina, sem Docker)

Abra um terminal por módulo e suba na ordem: brokers → drones → sensores (ou cliente).

```bash
# Brokers
python broker.py setor_a
python broker.py setor_b
python broker.py setor_c

# Drones
python drone.py drone_a1
python drone.py drone_b1
python drone.py drone_c1

# Sensores automáticos
python sensor.py setor_a
python sensor.py setor_b
python sensor.py setor_c

# OU cliente manual (substitui o sensor)
python cliente.py setor_a
```

Os defaults já apontam para `127.0.0.1` — nenhuma variável de ambiente é necessária para rodar localmente.

### Cliente manual

O cliente permite enviar ocorrências manualmente sem depender do sensor automático. É útil para testes controlados.

```bash
python cliente.py setor_a
```

```
Cliente manual — setor: setor_a
────────────────────────────────────────
Tipos de ocorrência:
  1 → averiguacao      (BAIXA)
  2 → possivel_ataque  (MÉDIA)
  3 → ataque_concreto  (ALTA)
  q → sair
────────────────────────────────────────

Digite o tipo de ocorrência [1/2/3/q]:
```

### Em múltiplas máquinas (laboratório)

Descubra o IP de cada máquina com `hostname -I` e passe via variável de ambiente no formato `IP:PORTA`.

```bash
# Máquina com setor_a (IP 192.168.1.10)
BROKER_A=192.168.1.10:5001 BROKER_B=192.168.1.11:5002 BROKER_C=192.168.1.12:5003 \
python broker.py setor_a

# Drone na mesma máquina
BROKER_A=192.168.1.10:6001 BROKER_B=192.168.1.11:6002 BROKER_C=192.168.1.12:6003 \
python drone.py drone_a1

# Sensor na mesma máquina
BROKER_A=192.168.1.10:7001 BROKER_B=192.168.1.11:7002 BROKER_C=192.168.1.12:7003 \
python sensor.py setor_a

# Cliente manual (pode rodar em qualquer máquina)
BROKER_A=192.168.1.10:7001 BROKER_B=192.168.1.11:7002 BROKER_C=192.168.1.12:7003 \
python cliente.py setor_b
```

> Drones, sensores e clientes podem rodar em qualquer máquina da rede — basta apontar as variáveis para os IPs corretos. A porta muda conforme o módulo: `5001–5003` para broker↔broker, `6001–6003` para drones, `7001–7003` para sensores e clientes.

### Com Docker

```bash
# Build das imagens
docker build -t lucasarguerra/redes2-broker:latest  ./broker
docker build -t lucasarguerra/redes2-drone:latest   ./drone
docker build -t lucasarguerra/redes2-sensor:latest  ./sensor
docker build -t lucasarguerra/redes2-cliente:latest ./cliente
docker build -t lucasarguerra/redes2-testes:latest  ./testes

# Push para o Docker Hub
docker push lucasarguerra/redes2-broker:latest
docker push lucasarguerra/redes2-drone:latest
docker push lucasarguerra/redes2-sensor:latest
docker push lucasarguerra/redes2-cliente:latest
docker push lucasarguerra/redes2-testes:latest

# Broker (--network host obrigatório para comunicação entre máquinas físicas)
docker run -d --network host \
  -e BROKER_A=<IP_A>:5001 -e BROKER_B=<IP_B>:5002 -e BROKER_C=<IP_C>:5003 \
  lucasarguerra/redes2-broker:latest python broker.py setor_a

# Drone
docker run -d --network host \
  -e BROKER_A=<IP_A>:6001 -e BROKER_B=<IP_B>:6002 -e BROKER_C=<IP_C>:6003 \
  lucasarguerra/redes2-drone:latest python drone.py drone_a1

# Sensor automático
docker run -d --network host \
  -e BROKER_A=<IP_A>:7001 -e BROKER_B=<IP_B>:7002 -e BROKER_C=<IP_C>:7003 \
  lucasarguerra/redes2-sensor:latest python sensor.py setor_a

# Cliente manual (-it obrigatório para input interativo)
docker run -it --network host \
  -e BROKER_A=<IP_A>:7001 -e BROKER_B=<IP_B>:7002 -e BROKER_C=<IP_C>:7003 \
  lucasarguerra/redes2-cliente:latest python cliente.py setor_a
```

Para ver os logs de um container rodando em background:
```bash
docker logs -f <ID_DO_CONTAINER>
docker ps  # lista containers ativos e seus IDs
```

---

## Testes

A pasta de testes contém três scripts para validar o sistema sob diferentes condições.

### `teste_carga.py` — ordenação da fila

Envia N requisições em ordem aleatória para setores aleatórios e imprime a ordem esperada na fila para comparação visual com os brokers.

```bash
python teste_carga.py 20
# ou via Docker (-it para input interativo)
docker run -it --network host lucasarguerra/redes2-testes:latest python teste_carga.py 20
```

### `teste_consistencia.py` — consistência entre brokers

Envia N requisições, aguarda propagação e imprime a distribuição esperada por prioridade para verificação manual nos terminais dos três brokers.

```bash
python teste_consistencia.py 15
docker run -it --network host lucasarguerra/redes2-testes:latest python teste_consistencia.py 15
```

### `teste_falhas.py` — condições críticas (menu interativo)

```bash
python teste_falhas.py
docker run -it --network host lucasarguerra/redes2-testes:latest python teste_falhas.py
```

Oferece quatro cenários:

| Cenário | O que testa |
|---------|-------------|
| Rajada concorrente | Exclusão mútua sob alta concorrência — todas as requisições disparadas ao mesmo tempo |
| Durante reconexão de broker | Consistência da fila após queda e retorno de um broker |
| Queda de drone durante missão | Recuperação de requisição quando drone cai no meio da missão |
| Stress test em ondas | Carga contínua em múltiplas ondas para avaliar estabilidade |

---

## Estrutura de Arquivos

```
.
├── broker/
│   ├── broker.py           # Lógica principal: fila, despacho, tolerância a falhas
│   ├── RicartAgrawala.py   # Exclusão mútua distribuída + roteador de mensagens
│   └── Dockerfile
├── drone/
│   ├── drone.py            # Cliente TCP: recebe missões, reporta conclusão
│   └── Dockerfile
├── sensor/
│   ├── sensor.py           # Gerador de ocorrências aleatórias
│   └── Dockerfile
├── cliente/
│   ├── cliente.py          # Interface manual para envio de ocorrências
│   └── Dockerfile
└── testes/
    ├── teste_carga.py        # Teste de ordenação por prioridade
    ├── teste_consistencia.py # Teste de replicação da fila entre brokers
    ├── teste_falhas.py       # Testes de condições críticas (menu interativo)
    └── Dockerfile
```
