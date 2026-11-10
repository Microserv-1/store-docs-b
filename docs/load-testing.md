# Testes de Carga, Escalabilidade e Revisão Cruzada

## Metodologia de Carga Reproduzível

O script `scripts/load.py` no repositório integrador `store-root` executa carga autenticada em modelo fechado (*closed-loop*, sem *think time*) através do **Gateway Go**, suportando todos os cenários da dupla:

- `identity`: leitura autenticada em `GET /accounts/me` (Pessoa A)
- `login`: autenticação intensiva em CPU (`Argon2id` + emissão JWT) em `POST /auth/login` (Pessoa A)
- `products_cached`: leitura autenticada em `GET /products/{id}` servida pelo cache Redis (`X-Cache: HIT`) (Pessoa B)
- `products_uncached`: leitura autenticada em `GET /products/{id}` forçando bypass ao PostgreSQL (`X-Cache-Control: no-cache`, `X-Cache: BYPASS`) (Pessoa B)
- `exchange`: cotação autenticada em `GET /exchanges/USD/BRL` (Pessoa B)
- `orders`: criação autenticada de pedidos em `POST /orders` com validação em `Product` e publicação assíncrona no `RabbitMQ` (Pessoa B)
- `business_mix`: carga mista combinando leituras de catálogo/produto (`GET /products`, `GET /products/{id}`), cotação de câmbio (`GET /exchanges/USD/BRL`) e criação de pedidos (`POST /orders`)

### Comandos de Execução

```bash
# Comparação controlada sob carga: Product com Redis vs sem Redis
python3 scripts/load.py --users 8 --duration 20 --scenario products_cached --output artifacts/load-products-cached.json
python3 scripts/load.py --users 8 --duration 20 --scenario products_uncached --output artifacts/load-products-uncached.json

# Carga de criação de pedidos (Order + Product + RabbitMQ)
python3 scripts/load.py --users 8 --duration 20 --scenario orders --output artifacts/load-orders.json

# Carga mista dos 6 serviços
python3 scripts/load.py --users 8 --duration 20 --scenario business_mix --output artifacts/load-business-mix.json
```

Cada relatório JSON registra automaticamente:
- Timestamp UTC, duração solicitada e duração real medida;
- Total de requisições, vazão (`requests_per_second`) e razão de erros (`error_ratio`);
- Percentis de latência (`p50`, `p95`, `p99`) em milissegundos;
- Contagem de status HTTP (`status_counts`) e de respostas de cache (`x_cache_counts`);
- Distribuição de instâncias (`upstream_instances` via cabeçalho `X-Served-By`);
- Hash exato do commit (`HEAD`) e estado da working tree (`dirty`) dos 6 microsserviços (`account`, `auth`, `gateway`, `product`, `order`, `exchange`).

---

## Revisão Cruzada da Pessoa B sobre as Entregas da Pessoa A

Conforme exigido pelo roteiro do projeto, a **Pessoa B (Gustavo Lima Affonso Ferreira)** revisou e executou os cenários dos serviços da **Pessoa A (Yuri Henrique)**:

| Item Revisado por B | Repositório / Componente de A | Verificação Executada por B | Resultado |
|---|---|---|---|
| Cadastro, unicidade de e-mail e hash Argon2id | `store-account` (`be893df`) | Execução de `tests/smoke.py` e `tests/persistence.py` validando `201`, `409` em e-mail duplicado case-insensitive e ausência de senha no payload | **Aprovado** |
| Login JWT HS256 e bloqueio da rota interna | `store-auth` (`e3fa400`) + `store-gateway` (`a8f2694`) | Validação de `POST /auth/login` (`200`), rejeição de senha incorreta (`401`), bloqueio de `/internal/credentials/verify` no Gateway (`404`) | **Aprovado** |
| Anti-spoofing de `X-Account-ID` no Gateway | `store-gateway` (`a8f2694`) | Tentativa de envio de `X-Account-ID` forjado com token de outra conta; Gateway sobrescreveu corretamente com o `sub` do JWT validado | **Aprovado** |
| Roteamento dos prefixos `/products`, `/orders`, `/exchanges` | `store-gateway` (`a8f2694`) | Integração real dos 3 serviços de B atrás do Gateway de A em Docker Compose e Kubernetes (`kind-microserv-store`) | **Aprovado** |

Os resultados numéricos consolidados das execuções em Docker Compose e Kubernetes estão documentados em [Estado e evidências](status.md).
