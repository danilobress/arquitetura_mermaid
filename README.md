# WaitList Pro - Documentação de Arquitetura

Este repositório serve como base de documentação arquitetural. Ele foi projetado para fornecer contexto claro e decisões consolidadas, permitindo auxilio na implementação de novas funcionalidades com total aderência ao design e arquitetura concebida.


---


## 1. Descrição do Sistema

### 📌 Escopo
O **WaitList Pro** é um sistema de gestão de filas de espera eletrônicas para restaurantes. O escopo principal é digitalizar e otimizar a experiência de espera por uma mesa:
- Entrada do cliente na fila via QR Code e acompanhamento em tempo real.
- Gestão operacional e orquestração de chamadas de mesas.
- Notificações ativas (SMS/WhatsApp) integradas ao fluxo de chamada.

### 🚫 Fora do Escopo
- Cardápio digital e integração com sistema de pedidos.
- Pagamentos e split de contas.
- Gestão de estoque ou controle financeiro do restaurante.
- Integração complexa com sistemas legados de PDV.

### 🔭 Nível da Visão
A documentação está estruturada no **Nível de Containers (C4 Model)** e no **Nível de Comportamento (Diagramas de Sequência)**. O foco é detalhar a topologia dos aplicativos clientes, a API responsável pelas regras de negócio, o banco de dados em memória para performance na fila e as integrações vitais.

### 🧱 Limites e Responsabilidades
- **Customer Web App (PWA):** Interface leve, voltada para a entrada na fila e atualização de status. Não retém lógica de negócio.
- **Hostess Tablet App:** Interface operacional que comanda a fila, aciona chamadas de clientes e finaliza atendimentos.
- **WaitList API:** Concentra regras de negócio, validações, cálculo de tempo estimado e a orquestração de transações.
- **Queue DB (Redis):** Fila ativa. Mantém os clientes aguardando e chamados. Responsável por garantir latência baixa para os painéis de acompanhamento.
- **Main DB (PostgreSQL):** Persistência. Armazena histórico, relatórios de abandono, tempo médio de espera (SLA) e auditoria de tickets finalizados.

### 🔌 Integrações Externas
- **Mensageria (Twilio / API WhatsApp):** Disparo de notificações ("Sua mesa está pronta") de forma assíncrona, para não travar a API, recebendo webhooks de status de entrega.
- **Provedor de Identidade (Auth):** Autenticação e SSO exclusivo para os colaboradores (Hostess/Gerentes) acessarem o sistema.

### 🚧 Restrições Técnicas
- Alta disponibilidade (99.9%) requerida durante os horários de pico (Sexta a Domingo à noite).
- Latência baixa para as operações de fila (leitura e escrita), obrigando o uso de banco em memória (Redis).
- Tolerância a falhas: o sistema não pode bloquear a Hostess se o PostgreSQL ou o disparo de SMS ficarem lentos.


---


## 2. Visão de Containers (C4 Model)

Para garantir escalabilidade na leitura em tempo real e resiliência, a arquitetura separa bancos de dados e utiliza comunicações assíncronas para saídas lentas.

👉 **[Ver Diagrama C4 (Flowchart)](./docs/arquitetura/c4-model.md)**


---


## 3. Comportamento e Fluxos (Diagramas de Sequência)

A separação entre Happy Path e Exception Paths seguiu o princípio de **Separation of Concerns**: cada diagrama conta uma história clara para facilitar a leitura e a manutenção.


### Diagrama A — Caminho Feliz (Happy Path)
Representa o fluxo principal: `AGUARDANDO → CHAMADO → ATENDIDO`.

👉 **[Ver Diagrama A: Caminho Feliz](./docs/arquitetura/sequence-happy-path.md)**


### Diagrama B — Tratamento de Exceções
Fluxo de Exceção: transições de exceção como `AGUARDANDO → DESISTIU` e `CHAMADO → NAO_COMPARECEU`.

👉 **[Ver Diagrama B: Tratamento de Exceções](./docs/arquitetura/sequence-exceptions.md)**


---


## 4. Utilização da GenAI

O auxílio na criação do diagrama C4 Model e diagramas de sequência, desta arquitetura foram realizados com o auxílio de IA Generativa atuando como **Agente Colaborador (Engenheiro / Arquiteto Sênior)**. 
- A IA auxiliou não apenas na geração rápida da sintaxe do código Mermaid (para o C4 Model e o Sequência), mas principalmente nas **questões analíticas de vulnerabilidades do fluxo**.
- A GenAI ajudou na criação dos diagramas simulando cenários de alta concorrência ("o que acontece se a requisição dobrar?") e sugeriu boas implementações. Porém foi necessário rever os pontos de melhoria aplicados posteriormente.
- O refinamento visual dos diagramas para acessibilidade, contraste adequado (Dark Theme) e clareza estrutural foi gerado mediante prompts focados em design de documentação. Porém a GenAI teve dificuldade visual de gerar por exemplo o C4 Model, o que foi preciso diferentes iterações para chegar no resultado atual

Com isso é possível observar que a GenAI é uma ferramenta de grande auxílio na criação de documentação, mas precisa de supervisão humana para garantir a qualidade do resultado final.


---


## 5. Decisões e Melhorias Aplicadas

Durante o desenho inicial gerado da arquitetura, algumas decisões foram tomadas após revisão com auxílio de IA generativa para melhorar a arquitetura proposta. As principais melhorias aplicadas foram:

1. **Substituição do motor C4Container pelo Flowchart (`graph TB`):**
   * **Motivo:** O motor padrão C4 cruzava setas por cima de containers, o que dificultava a visualização. O ajuste aplicado foi utilizar a renderização com fluxo Top-Down (Atores → Apresentação → API → Bancos → Externos).
2. **Persistência Assíncrona no PostgreSQL:**
   * **Motivo e Melhoria:** Para não travar o processamento, o ciclo vivo ocorre inteiro no Redis. Apenas o armazenamento no PostgreSQL é feito via eventos disparados de forma assíncrona.
3. **Separação de Fluxos (Happy Path e Exceções):**
   * **Motivo e Melhoria:** Para garantir clareza na documentação, separamos os fluxos em dois diagramas.
   * **Happy Path:** Fluxo ideal de negócio: `AGUARDANDO → CHAMADO → ATENDIDO`.
   * **Exceções:** Transições de exceção como `AGUARDANDO → DESISTIU` e `CHAMADO → NAO_COMPARECEU`.


---


## 6. Lacunas

Apesar da documentação apresentar uma visão completa do sistema em C4 e diagramas de sequência, ainda existem diversas informações, regras de negócio e definições técnicas que precisam ser detalhadas antes da implementação. Entre elas:

- Qual será o algoritmo ou fórmula exata para calcular o tempo estimado de espera para o cliente?
- Como será feito o controle de limite de requisições (rate limit) no QR Code para evitar ataques de spam ou DDoS na fila?
- O que acontece com a fila atual se o banco em memória (Redis) sofrer um *restart* inesperado?
- Qual mecanismo exato será utilizado para a comunicação em tempo real entre os apps e o backend?
- Quantas posições na fila um mesmo número de celular (cliente) pode ocupar simultaneamente?
- Qual o padrão estrutural dos payloads (JSON) e schemas da API na comunicação entre os apps e o backend?


---


## 7. O que seria necessário para um agente construir o sistema sem inventar decisões?

Embora os diagramas C4 e de Sequência ofereçam uma visão arquitetural completa, para que um agente autônomo escrevesse o código de ponta a ponta sem "alucinar" ou inventar lógicas, a documentação precisaria evoluir para o Nível 3 (Componentes) e Nível 4 (Código). Os itens que deveriam ter na documentação para isso seriam:

- **Contratos de API (OpenAPI/Swagger):** Definição estrita das rotas, métodos HTTP, payloads (JSON) e códigos de status de retorno.
- **Esquemas de Banco de Dados:** Um modelo Entidade-Relacionamento (ERD) detalhando tabelas, colunas, tipos de dados, chaves primárias/estrangeiras e índices.
- **Regras de Negócio Matemáticas:** Regras de negócio no geral, como por exemplo o sistema calcula o tempo estimado de espera. Sem isso, a GenIA poderia inventar um cálculo.
- **Design System / UI:** Mapeamento exato de componentes visuais, tokens de cores e layouts (ex: referências do Figma) para que o Customer Web App e o Tablet App não fossem gerados com interfaces genéricas.


---


## 8. Conclusão

A arquitetura do WaitList Pro (sistema de gerenciamento de filas de espera eletrônicas para restaurantes) foi pensada para balancear a necessidade de alta disponibilidade, resposta em tempo real e resiliência, garantindo uma boa experiência ao usuário final. O uso de IA generativa como auxílio provou ser uma abordagem importante para desenhar os diagramas e auxiliar na reflexão crítica sobre as decisões arquiteturais, mitigando riscos antes mesmo da implementação. Porém é crucial que um humano faça a revisão final para validar as decisões propostas pela IA e aplicar ajustes finos, pois a IA ainda tem limitações em compreender contextos específicos e regras de negócio complexas.
