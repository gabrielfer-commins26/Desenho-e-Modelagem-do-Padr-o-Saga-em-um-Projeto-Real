# Matriz de Decisão, Idempotência e Resiliência

Para garantir que o Padrão Saga funcione com segurança sob condições adversas de rede, as seguintes diretrizes arquiteturais devem ser implementadas no código:

## 1. Garantia de Idempotência (Idempotent Consumer)
Em sistemas assíncronos baseados em mensageria, mensagens podem ser entregues mais de uma vez (conceito de *at-least-once delivery*). Para evitar que um cliente pague duas vezes ou que o estoque seja descontado em dobro:
* **Correlation ID / Saga ID:** Toda transação iniciada pelo `Order Service` receberá um identificador único de Saga gerado via UUID.
* **Tabela de Controle:** Os microsserviços (`Inventory` e `Payment`) possuirão uma tabela/coleção em seus bancos de dados locais chamada `processed_messages`. Antes de processar um comando, o serviço verifica se aquele `Saga ID` + `Message ID` já foi processado. Se sim, ele descarta a ação repetida e apenas reenvia o evento de sucesso para o broker (Idempotência).

## 2. Tratamento de Timeouts (Máquina de Estados)
Se o `Order Service` enviar um comando `ProcessPayment` e o gateway financeiro demorar muito para responder:
* A Saga não pode ficar travada infinitamente no status `PENDING`.
* O `Order Service` implementará um temporizador (ex: 3 minutos). Se não houver retorno do evento via Message Broker neste prazo, o Orquestrador assume que houve uma falha de timeout, aborta a transação e dispara o evento de compensação (`ReleaseInventory`) para os passos anteriores.

## 3. Padrão Retry (Retentativas) e Circuit Breaker
Para lidar com instabilidades passageiras:
* **Retry com Exponential Backoff:** Se um serviço falhar ao ler do banco ou chamar uma API externa, o consumidor da mensagem tentará novamente de forma espaçada (ex: tenta em 2s, 4s, 8s).
* **Circuit Breaker (Disjuntor):** Se o `Payment Service` detectar que o Gateway do Banco caiu completamente, o disjuntor "abre". Em vez de acumular milhares de mensagens e sobrecarregar o sistema com *Retries* inúteis, o serviço começa a rejeitar as mensagens imediatamente (*Fail-fast*), permitindo que a Saga dispare a compensação mais rápido.

## 4. Dead-Letter Queue (DLQ) e Falhas de Compensação
A maior preocupação na Saga é: *"O que acontece se a própria transação de compensação falhar?"* (Ex: O sistema não consegue devolver o estoque de jeito nenhum).
* Não é possível desfazer uma compensação. Portanto, comandos de compensação devem tentar execução até terem sucesso.
* Caso excedam o número máximo de retentativas (problema grave de banco de dados, por exemplo), a mensagem será movida para uma **Dead-Letter Queue (DLQ)**.
* A DLQ acionará um alerta (via Grafana/Slack) para a equipe de SRE (Site Reliability Engineering), que precisará intervir manualmente para corrigir a inconsistência nos dados de estoque.
