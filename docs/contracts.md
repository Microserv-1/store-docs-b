# Contratos e Integração dos Serviços da Pessoa B

Todos os serviços da Pessoa B seguem rigorosamente os contratos de borda definidos no Gateway (`store-gateway`) e no serviço de autenticação (`store-auth`), escutando na porta `8000` dentro da rede interna e validando o token JWT `HS256` (`iss="store-auth"`, `aud="store-api"`, `sub=<account_uuid>`).

## Cabeçalhos e Rotas Operacionais Comuns

Todos os 3 serviços (`product`, `order`, `exchange`) implementam:

- `GET /health-check` -> `200 {"status": "ok", "service": "<nome>"}`
- `GET /ready` -> `200` quando dependências críticas (PostgreSQL/Redis/RabbitMQ) estão acessíveis, ou `503` caso contrário
- `GET /metrics` -> métricas Prometheus (`http_requests_total`, histogramas de latência e métricas customizadas de cache/mensageria/câmbio)
- Cabeçalhos de resposta em todas as chamadas: `X-Request-ID` e `X-Served-By` (identificando o pod/container que processou a requisição)

---

## 1. Product Service (`Microserv-1/store-product`)

Responsável pelo catálogo de produtos em **USD** e pelo tratamento do **Gargalo B1 (Cache Redis)**.

### Endpoints

| Método | Rota no Gateway | Status | Descrição |
|---|---|---|---|
| `POST` | `/products` | `201 Created` | Cria produto no PostgreSQL e invalida chaves de listagem no Redis |
| `GET` | `/products` | `200 OK` | Lista catálogo (suporta filtro `?name=` com `ILIKE` case-insensitive) usando Cache-Aside no Redis |
| `GET` | `/products/{id}` | `200 OK` / `404` | Consulta produto por UUID usando Cache-Aside no Redis (`X-Cache: HIT\|MISS\|BYPASS`) |
| `DELETE` | `/products/{id}` | `204 No Content` / `404` | Remove produto do PostgreSQL e invalida imediatamente o item e as listagens no Redis |
| `GET` | `/products/cache/stats` | `200 OK` | Retorna contadores de `hits`, `misses`, `invalidations`, `bypasses`, `hit_ratio` e `ttl_seconds` |

### Exemplo de Payload e Resposta (`POST /products`)

Requisição:
```json
{
  "name": "Cloud Server M5",
  "price": 120.50,
  "unit": "month",
  "stock": 15
}
```

Resposta (`201 Created`):
```json
{
  "id": "8e105a9a-4f08-4f5a-8a3f-9d8a22f1d101",
  "name": "Cloud Server M5",
  "price": 120.5,
  "unit": "month",
  "stock": 15,
  "created_at": "2026-11-10T22:15:00Z"
}
```

---

## 2. Exchange Service (`Microserv-1/store-exchange` — Python Obrigatório)

Microsserviço escrito obrigatoriamente em **Python 3.12 / FastAPI**, responsável pela conversão monetária em tempo real consultando provedores externos públicos com fallback resiliente:

1. **Provedor Primário**: `AwesomeAPI` (`https://economia.awesomeapi.com.br/json/last/{FROM}-{TO}`)
2. **Provedor Secundário (Fallback)**: `Open ExchangeRate-API` (`https://open.er-api.com/v6/latest/{FROM}`) com spread configurável (`EXCHANGE_SPREAD_BPS`, padrão `50` bps = `0,5%`)
3. **Cache em memória com TTL** (`CACHE_TTL_SECONDS=60`) e fallback stale caso ambos os provedores externos sofram indisponibilidade momentânea.

### Endpoints

| Método | Rota no Gateway | Status | Descrição |
|---|---|---|---|
| `GET` | `/exchanges/{from}/{to}` | `200 OK` | Retorna cotação `sell`, `buy`, `date` e `id-account` extraído do JWT |
| `GET` | `/exchanges/{from}/{to}` | `422 Unprocessable Entity` | Moeda não suportada (ex.: `INVALID`) |
| `GET` | `/exchanges/{from}/{to}` | `503 Service Unavailable` | Provedores externos indisponíveis e sem cotação em cache |

### Exemplo de Resposta (`GET /exchanges/USD/BRL`)

```json
{
  "sell": 5.7412,
  "buy": 5.7358,
  "date": "2026-11-10 19:20:00",
  "id-account": "3b99f080-51df-4209-87bc-02f1b2467c10"
}
```

---

## 3. Order Service (`Microserv-1/store-order`)

Responsável pela criação e consulta de pedidos vinculados ao `id_account` autenticado no JWT, integração síncrona com `Product` e `Exchange`, e processamento assíncrono via **RabbitMQ (Gargalo B2)**.

### Regras de Negócio Verificadas

- **Validação de Produtos**: Em `POST /orders`, cada `id_product` é consultado em `store-product` repassando o header `Authorization: Bearer <token>`. Se qualquer produto não existir (`404`), o pedido é rejeitado com `400 Bad Request`.
- **Armazenamento Canônico em USD**: O preço unitário e o total do pedido são calculados com `Decimal` (`ROUND_HALF_UP`, 2 casas decimais) e persistidos em **USD** no PostgreSQL.
- **Isolamento por Usuário**: `GET /orders` retorna apenas os pedidos cujo `id_account` coincide com o `sub` do JWT. `GET /orders/{id}` retorna `404 Not Found` caso o pedido pertença a outra conta.
- **Conversão On-the-Fly sem Mutação**: `GET /orders/{id}?currency=BRL` consulta `GET /exchanges/USD/BRL`, aplica a taxa `sell` sobre os itens e sobre o total com arredondamento `ROUND_HALF_UP` (2 casas decimais) e retorna `currency="BRL"` e `exchange_rate=<sell>`, **sem alterar o valor em USD salvo no banco**. Moedas inválidas retornam `422`.
- **Mensageria Assíncrona e Idempotência**: `POST /orders` grava o evento `order.created` em tabela outbox (`order_outbox_events`) e publica na fila durável `order.events` do RabbitMQ (com Dead-Letter Queue `order.events.dlq`). O worker consumidor processa o evento de forma **idempotente por `order_id`**, registrando a notificação em `order_notifications` sem duplicar linhas em caso de reentrega.

### Endpoints

| Método | Rota no Gateway | Status | Descrição |
|---|---|---|---|
| `POST` | `/orders` | `201 Created` / `400` | Valida produtos, persiste pedido em USD e publica evento `order.created` no RabbitMQ |
| `GET` | `/orders` | `200 OK` | Lista pedidos exclusivamente da conta autenticada |
| `GET` | `/orders/{id}?currency=USD` | `200 OK` / `404` / `422` | Detalha pedido (em USD por padrão ou convertido via `Exchange` quando `?currency=` é informado) |
| `GET` | `/orders/{id}/notifications` | `200 OK` / `404` | Lista notificações assíncronas processadas pelo consumidor RabbitMQ para o pedido |
| `POST` | `/orders/{id}/events/replay` | `200 OK` / `404` | Republica o evento `order.created` na fila RabbitMQ para demonstrar idempotência do consumidor |
| `GET` | `/orders/events/summary` | `200 OK` | Resumo operacional da fila (`outbox_published`, `notifications_processed`, `queue_depth`, estado do consumidor) |
| `POST` | `/orders/events/consumer` | `200 OK` | Pausa (`{"enabled": false}`) ou retoma (`{"enabled": true}`) o consumidor interno para demonstração de resiliência |
