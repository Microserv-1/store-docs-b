# Gargalos da Pessoa B: Cache Redis (B1) e Mensageria RabbitMQ (B2)

Conforme o roteiro oficial de divisão em dupla, a **Pessoa B** é responsável pela implementação, instrumentação e comprovação experimental de dois tratamentos de gargalos arquiteturais nos microsserviços de negócio:

1. **Tratamento B1 — Cache Redis (`Cache-Aside`) no `Product Service`**
2. **Tratamento B2 — Mensageria Assíncrona e Idempotência com RabbitMQ no `Order Service`**

---

## 1. Tratamento B1: Cache Redis no Product Service (`store-product`)

### Problema de Arquitetura
Em um e-commerce, a leitura de catálogo (`GET /products`) e de detalhes de produtos (`GET /products/{id}`) ocorre em frequência muito superior à escrita, além de `GET /products/{id}` ser chamado sincronicamente pelo `Order Service` a cada item de cada pedido criado. Sem cache, toda leitura atinge o PostgreSQL, elevando a latência de cauda (`p95`/`p99`) e disputando conexões com transações de escrita (`POST /orders`, `POST /accounts`).

### Solução Implementada
Implementamos o padrão **Cache-Aside** usando **Redis 7** no `store-product`:
- **Chaves de Cache**:
  - Detalhe de produto: `products:id:<uuid>`
  - Listagem: `products:list:all`; busca: `products:list:search:<filtro_normalizado>`
- **Fluxo de Leitura (`GET /products` e `GET /products/{id}`)**:
  1. Consulta a chave no Redis. Se encontrada, retorna imediatamente com cabeçalho `X-Cache: HIT` e incrementa `product_cache_hits_total{operation="get_by_id|list"}`.
  2. Se ausente (`X-Cache: MISS`), consulta o PostgreSQL, serializa o resultado no Redis com `EX = CACHE_TTL_SECONDS` (padrão `120s`) e retorna a resposta.
  3. Suporta `?cache=bypass` e cabeçalho de controle `X-Cache-Control: no-cache` (`X-Cache: BYPASS`) para permitir comparação controlada A/B no mesmo ambiente sem desligar o Redis.
- **Política de Expiração e Invalidação em Escrita/Remoção**:
  - Em `POST /products`: invalida imediatamente todas as chaves `products:list:*` para que o novo produto apareça nas listagens sem esperar o TTL.
  - Em `DELETE /products/{id}`: remove o registro no PostgreSQL e invalida imediatamente `products:id:<id>` e `products:list:*`. Em sequência normal, uma leitura subsequente retorna `404 Not Found`. Leituras concorrentes e invalidações perdidas durante falha do Redis podem manter dados obsoletos até o TTL; cache-aside não oferece consistência forte.
  - Caso o Redis fique indisponível, o serviço degrada graciosamente para leitura direta no PostgreSQL (`X-Cache: BYPASS`) sem derrubar a API.

```mermaid
sequenceDiagram
    participant C as Cliente / Order Service
    participant G as Gateway Go
    participant P as Product Service
    participant R as Redis Cache
    participant DB as PostgreSQL

    C->>G: GET /products/{id} (Bearer JWT)
    G->>P: GET /products/{id}
    P->>R: GET products:id:{id}
    alt Cache HIT
        R-->>P: JSON payload
        P-->>G: 200 OK (X-Cache: HIT)
    else Cache MISS
        R-->>P: nil
        P->>DB: SELECT * FROM products WHERE id = {id}
        DB-->>P: Row
        P->>R: SETEX products:id:{id} 120 JSON
        P-->>G: 200 OK (X-Cache: MISS)
    end
    G-->>C: 200 OK
```

### Como Reproduzir o Experimento B1
```bash
python3 tests/cache_bottleneck.py --users 8 --requests-per-phase 240 --output artifacts/cache-bottleneck.json
```
O script valida:
1. Leituras com `MISS`/`HIT` (o POST já pode popular o detalhe no cache);
2. Invalidação imediata após `DELETE /products/{id}` (`204` -> `404`);
3. Benchmark comparativo de vazão (`req/s`) e latência (`avg`, `p50`, `p95`, `p99`) com e sem cache para `GET /products/{id}` e `GET /products`.

---

## 2. Tratamento B2: Mensageria Assíncrona com RabbitMQ no Order Service (`store-order`)

### Problema de Arquitetura
Quando um pedido é criado (`POST /orders`), tarefas posteriores como geração de notificações ao cliente, auditoria de eventos e integrações de backoffice não devem bloquear a resposta HTTP do checkout nem causar falha da compra caso o processador de notificações esteja temporariamente lento ou indisponível.

### Solução Implementada
Implementamos desacoplamento assíncrono com **Transactional Outbox + RabbitMQ 3 + Consumidor Idempotente**:
- **Publicação Confiável (Outbox + Fila Durável)**:
  - Ao criar um pedido em `POST /orders`, o `Order Service` persiste na mesma transação do banco o pedido (`orders`), seus itens (`order_items`) e o evento `order.created` na tabela `order_outbox_events` (`status="PENDING"`).
  - Em seguida, publica a mensagem com `delivery_mode=2` (persistente) na fila durável `order.events` do RabbitMQ (configurada com Dead-Letter Queue `order.events.dlq`) com `confirm_delivery()` e `mandatory=True`; somente após confirmação marca `published=True` na outbox. Uma falha após publicação e antes do commit pode causar duplicatas. Se o broker estiver momentaneamente indisponível no instante do `POST`, a thread de background republica automaticamente os eventos `PENDING` assim que a conexão AMQP é restabelecida.
- **Consumo Assíncrono com Controle de Resiliência**:
  - O worker consumidor lê mensagens da fila `order.events` com `basic_get(auto_ack=False)` e só envia `basic_ack` após a transação de notificação. Falhas geram `basic_nack(requeue=True)`. Payload inválido é publicado na DLQ; se essa publicação falhar, a mensagem original permanece recuperável. O contrato é de entrega pelo menos uma vez com consumidor idempotente, sem garantia de exatamente uma vez.
  - O endpoint operacional `POST /orders/events/consumer` (`{"enabled": false|true}`) retorna 404 por padrão, sem alterar o estado. Somente com `ENABLE_DEMO_CONTROLS=true`, em demonstração controlada, permite simular a queda/pausa do worker de notificações enquanto novos pedidos continuam sendo criados normalmente com baixa latência.
- **Garantia de Idempotência**:
  - A tabela `order_notifications` possui restrição `UNIQUE (order_id)` no PostgreSQL.
  - Quando um mesmo evento `order.created` é entregue mais de uma vez (at-least-once delivery ou via `POST /orders/{id}/events/replay`), o consumidor detecta o registro existente para aquele `order_id`, incrementa apenas o contador `duplicate_deliveries` e confirma a mensagem (`ACK`) **sem duplicar a notificação**.

```mermaid
sequenceDiagram
    participant C as Cliente
    participant O as Order API
    participant DB as PostgreSQL (orders + outbox + notifications)
    participant MQ as RabbitMQ (order.events)
    participant W as Worker Consumidor (Idempotente)

    C->>O: POST /orders (Bearer JWT)
    O->>DB: INSERT orders, order_items, order_outbox_events
    O->>MQ: basic_publish(queue="order.events", persistent)
    O-->>C: 201 Created (resposta imediata em USD)
    MQ->>W: Entrega evento order.created (order_id, event_id)
    W->>DB: Upsert idempotente por order_id em order_notifications
    W->>MQ: basic_ack
```

### Como Reproduzir o Experimento B2
```bash
python3 tests/messaging_bottleneck.py --batch-size 12 --output artifacts/messaging-bottleneck.json
```
O experimento histórico exige opt-in de demonstração e não substitui o aceite atual `business_integration.py`/`business_resilience.py`. O script verifica:
1. Com o consumidor pausado (`consumer_enabled: false`), 12 chamadas `POST /orders` respondem `201 Created` rapidamente enquanto as mensagens ficam retidas com segurança;
2. Ao reativar o consumidor (`consumer_enabled: true`), todas as 12 notificações acumuladas são drenadas e processadas em milissegundos;
3. Ao republicar 2 vezes o evento de um pedido já processado (`POST /orders/{id}/events/replay`), a contagem de linhas em `GET /orders/{id}/notifications` permanece igual a `1`, enquanto `duplicate_deliveries` sobe para `2`, provando idempotência.
