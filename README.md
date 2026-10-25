# Blueprint de Arquitetura: Padrão Saga no E-commerce

## 1. Contexto de Negócios
Este repositório contém a documentação arquitetural da transição de um monólito de comércio eletrônico para uma arquitetura de microsserviços. O fluxo de checkout engloba três domínios críticos: **Order Service**, **Inventory Service** e **Payment Service**. Devido à natureza distribuída do sistema, transações ACID convencionais não são aplicáveis. O padrão Saga foi adotado para garantir a consistência eventual e gerenciar rollbacks lógicos (compensações) em caso de falhas.

## 2. Decisão Arquitetural: O Estilo da Saga
**Escolha:** Optou-se pelo modelo de **Orquestração** (Saga Orchestration).

**Justificativa:**
* **Acoplamento e Complexidade:** No modelo de Coreografia, os serviços escutam eventos uns dos outros, o que gera um acoplamento implícito. Se o fluxo crescer (ex: adicionar serviço de Nota Fiscal e Entrega), a coreografia se torna um "labirinto de eventos" difícil de rastrear. Na Orquestração, o `Order Service` atua como o cérebro (Máquina de Estados), conhecendo todo o fluxo, enquanto `Inventory` e `Payment` continuam "cegos" sobre o negócio global (apenas respondem a comandos).
* **Escalabilidade:** A orquestração permite fácil inserção de novos passos no fluxo no futuro, bastando atualizar o orquestrador.

## 3. Descrição dos Diagramas
Os desenhos de arquitetura encontram-se na pasta `/diagrams/`:
* `saga-architecture-overview.png`: Visão estrutural mostrando os 3 microsserviços integrados de forma assíncrona através de um Message Broker (como Kafka/RabbitMQ).
* `sequence-success-flow.png`: O "Caminho Feliz". O orquestrador solicita a reserva do estoque e, após o sucesso, solicita a cobrança ao serviço de pagamentos, concluindo o pedido.
* `sequence-compensation-flow.png`: O cenário de falha. O estoque é reservado, mas o pagamento é negado. O orquestrador age rapidamente emitindo um comando de compensação (`ReleaseInventory`) para desfazer a reserva de estoque, garantindo que o item volte para a prateleira virtual.
