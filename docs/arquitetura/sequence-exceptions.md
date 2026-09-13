# Diagrama B — Tratamento de Exceções

Este diagrama documenta os **caminhos alternativos e de erro** do sistema: desistência voluntária do cliente, abandono por timeout, e falha no envio de notificações. Representa as transições de exceção (`AGUARDANDO → DESISTIU` e `CHAMADO → NAO_COMPARECEU`).

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
    note over C, WA: EXCEÇÃO 1 — Cliente Desiste Voluntariamente
    %% ══════════════════════════════════════════════════════════

    note right of WA: Pré-condição: Cliente com status AGUARDANDO

    C->>WA: Clica em "Desistir" ou fecha a aba
    WA->>API: DELETE /queue/{clienteId}

    API->>Redis: MULTI (transação atômica)
    API->>Redis: ZREM fila_espera {clienteId}
    API->>Redis: HSET ticket:telefone:{hash} status "DESISTIU"
    API->>Redis: EXEC
    Redis-->>API: OK — Cliente removido da fila

    API-)DB: INSERT historico_atendimento (status=DESISTIU, timestamp)

    API-->>WA: 200 OK — Removido da fila
    WA-->>C: Exibe confirmação de saída

    note right of WA: O cliente pode entrar novamente na fila (o ticket ativo foi encerrado)

    %% ══════════════════════════════════════════════════════════
    note over Msg, API: EXCEÇÃO 2 — Falha na Entrega de Notificação
    %% ══════════════════════════════════════════════════════════

    note right of Msg: Pré-condição: API disparou POST /notify na Fase 2

    Msg-)API: POST /webhooks/messaging (status: failed)
    note right of API: Número inválido, WhatsApp desativado ou timeout

    API-)HT: Evento SSE — "Notificação falhou para {nome}"
    HT-->>H: Exibe alerta: chamar cliente no viva-voz do restaurante

    %% ══════════════════════════════════════════════════════════
    note over API, Redis: EXCEÇÃO 3 — Abandono por Timeout (Worker)
    %% ══════════════════════════════════════════════════════════

    note right of API: Pré-condição: Cliente com status CHAMADO há mais de 5 minutos

    note right of API: Worker agendado (cron a cada 30s) verifica fila_chamados
    API->>Redis: ZRANGEBYSCORE fila_chamados -inf {agora - 300s}
    Redis-->>API: Lista de clientes com chamada expirada

    API->>Redis: MULTI (transação atômica)
    API->>Redis: ZREM fila_chamados {clienteId}
    API->>Redis: HSET ticket:{clienteId} status "NAO_COMPARECEU"
    API->>Redis: EXEC
    Redis-->>API: OK — Cliente removido da fila de chamados

    API-)DB: INSERT historico_atendimento (status=NAO_COMPARECEU, timestamp)

    API-)HT: Evento SSE/WebSocket — "Chamada expirada para {nome}"
    HT-->>H: Move cliente para aba "Não Compareceu" com alerta visual
```
