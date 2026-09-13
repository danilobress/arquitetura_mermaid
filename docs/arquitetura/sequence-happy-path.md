# Diagrama de Sequência A — Caminho Feliz (Happy Path)

Este diagrama documenta o **ciclo de vida bem-sucedido** de um ticket de fila: o cliente entra, a hostess chama, e o cliente comparece para ser atendido. Representa o fluxo principal do sistema (`AGUARDANDO → CHAMADO → ATENDIDO`).

```mermaid
sequenceDiagram
    autonumber
    actor C as Cliente
    participant WA as Customer Web App
    participant API as WaitList API
    participant Redis as Queue DB (Redis)
    participant DB as Main DB (PostgreSQL)
    participant Msg as Gateway Mensageria
    participant HT as Hostess Tablet App
    actor H as Hostess

    note over API, HT: 🔐 Todas as chamadas da Hostess carregam JWT via Bearer Token

    %% ══════════════════════════════════════════════════════════
    note over C, Redis: FASE 1 — Cliente Entrando na Fila
    %% ══════════════════════════════════════════════════════════

    C->>WA: Escaneia QR Code e acessa o App
    C->>WA: Preenche dados (Nome, Tel, Grupo) e clica em "Entrar"

    note right of WA: ⚡ Idempotência por status ativo (não por lock cego)

    WA->>API: POST /queue/join (Idempotency-Key: hash(telefone+data))
    API->>Redis: HGET ticket:telefone:{hash} status

    alt Ticket ATIVO existe (status = AGUARDANDO ou CHAMADO)
        Redis-->>API: Status do ticket ativo encontrado
        API-->>WA: 200 OK — Retorna ticket existente (posição e tempo)
        WA-->>C: Exibe status da fila (você já está na posição X)

    else Sem ticket ativo (novo ou já finalizado)
        Redis-->>API: nil — nenhum ticket ativo

        API->>Redis: GET cache:media_rotatividade
        Redis-->>API: Média de rotatividade (pré-calculada a partir do histórico)
        API->>API: Calcula tempo estimado com base na média e tamanho da fila

        API->>Redis: MULTI (transação atômica)
        API->>Redis: HSET ticket:telefone:{hash} status "AGUARDANDO" nome {nome} grupo {n}
        API->>Redis: ZADD fila_espera {timestamp_entrada} {clienteId}
        API->>Redis: EXEC
        Redis-->>API: OK — Ticket criado e cliente na fila

        API-->>WA: 201 Created — Ticket, posição e tempo estimado
        WA-->>C: Exibe tela de acompanhamento em tempo real
    end

    C-)WA: [Polling / SSE] Atualiza posição periodicamente

    %% ══════════════════════════════════════════════════════════
    note over HT, H: FASE 2 — Hostess Chamando a Mesa
    %% ══════════════════════════════════════════════════════════

    H->>HT: Monitora painel de gestão da fila
    HT->>API: GET /queue?limit=5 (Authorization: Bearer JWT)
    API->>Redis: ZRANGE fila_espera 0 4 WITHSCORES
    Redis-->>API: Lista dos próximos clientes ordenada
    API-->>HT: Retorna lista (nome, grupo, tempo esperando)
    HT-->>H: Atualiza dashboard em tempo real

    H->>HT: Clica em "Chamar Cliente" para o próximo da fila
    HT->>API: POST /queue/{clienteId}/call

    note right of API: ⏱️ Transição atômica: AGUARDANDO → CHAMADO

    API->>Redis: MULTI (transação atômica)
    API->>Redis: ZREM fila_espera {clienteId}
    API->>Redis: ZADD fila_chamados {timestamp_chamada} {clienteId}
    API->>Redis: HSET ticket:{clienteId} status "CHAMADO"
    API->>Redis: EXEC
    Redis-->>API: OK — Transição atômica concluída

    par Disparo assíncrono — não bloqueia o retorno à Hostess
        API-)Msg: POST /notify — payload com nome e link do cliente
        Msg-)C: SMS/WhatsApp: "Sua mesa está pronta! Você tem 5 min."
    end

    API-->>HT: 200 OK — Cliente chamado com sucesso
    HT-->>H: Inicia contagem regressiva de 5 minutos na tela

    %% ══════════════════════════════════════════════════════════
    note over HT, H: FASE 2.5 — Cliente Comparece com Sucesso
    %% ══════════════════════════════════════════════════════════

    H->>HT: Cliente chegou — clica em "Confirmar Presença"
    HT->>API: POST /queue/{clienteId}/confirm

    API->>Redis: MULTI (transação atômica)
    API->>Redis: ZREM fila_chamados {clienteId}
    API->>Redis: HSET ticket:{clienteId} status "ATENDIDO"
    API->>Redis: EXEC
    Redis-->>API: OK — Cliente atendido

    API-)DB: INSERT historico_atendimento (status=ATENDIDO, tempo_espera, grupo)
    API-)Redis: Recalcula cache:media_rotatividade (com novo ponto de dado)

    API-->>HT: 200 OK — Mesa alocada
    HT-->>H: Remove cliente do painel e atualiza contadores
```
