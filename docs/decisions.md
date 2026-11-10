# Decisões Técnicas da Pessoa B e Declaração de Uso de IA

## 1. Registro de Decisões de Arquitetura (ADR Resumido — Pessoa B)

| Decisão | Justificativa Técnica | Trade-off / Limite Conhecido |
|---|---|---|
| **`store-exchange` em Python 3.12 / FastAPI** | Cumpre a restrição obrigatória do projeto (Exchange em Python) mantendo padronização de instrumentação Prometheus e validação JWT com os demais serviços Python | Em chamadas sem cache, depende da latência de rede das APIs externas de câmbio |
| **Duplo provedor externo no Exchange (`AwesomeAPI` + fallback `Open ER-API`) + cache com TTL** | Evita que indisponibilidade ou rate-limit de uma única API pública derrube a conversão monetária de pedidos (`GET /orders/{id}?currency=...`) | Se ambos os provedores externos estiverem inacessíveis e o cache estiver frio, retorna `503 Service Unavailable` |
| **Armazenamento canônico de pedidos em USD e conversão *on-the-fly* na leitura** | Garante consistência contábil em `orders` e `order_items` usando `Decimal` (`ROUND_HALF_UP`), permitindo consultar o mesmo pedido em `BRL`, `EUR`, `GBP` etc. sem mutar o valor histórico salvo | A taxa exibida em `?currency=BRL` reflete a cotação vigente no momento da consulta (cacheada por 60s no Exchange) |
| **Cache-Aside no Redis com invalidação ativa e suporte a `X-Cache-Control: no-cache`** | Reduz drasticamente a latência e a carga sobre o PostgreSQL nas rotas de leitura de produtos, evita dados obsoletos após `POST`/`DELETE` e permite benchmark controlado A/B no mesmo cluster | A invalidação usa scan de prefixo `product:list:*`, adequado para cardinalidade moderada de filtros |
| **Transactional Outbox + RabbitMQ + Idempotência por `order_id` no Order Service** | Garante que o evento `order.created` não seja perdido se o broker oscilar durante o `POST /orders` e assegura que reentregas da fila não gerem notificações duplicadas | O worker consumidor roda como thread dedicada em cada réplica do `store-order` (compartilhando recursos do container) |
| **Isolamento de repositórios com submódulos Git em `store-root`** | Cada microsserviço (`store-product`, `store-order`, `store-exchange`) tem ciclo de vida, testes, `Dockerfile`, `Jenkinsfile` e `k8s/k8s.yaml` independentes, enquanto `store-root` fixa os commits exatos | Exige publicar os commits nos repositórios filhos antes de atualizar o ponteiro na raiz |

---

## 2. Declaração Transparente sobre Uso de IA

Ferramentas de assistência por IA (Antigravity / modelos LLM) foram utilizadas durante o desenvolvimento das entregas da Pessoa B para acelerar a escrita de código dos serviços (`store-product`, `store-order`, `store-exchange`), manifestos Kubernetes/Compose, scripts de teste de gargalos (`tests/cache_bottleneck.py`, `tests/messaging_bottleneck.py`), automação de carga e estruturação desta documentação.

Diretrizes seguidas pelo integrante **Gustavo Lima Affonso Ferreira (Pessoa B)**:
1. **Verificação Empírica Real**: Nenhum resultado numérico, relatório JSON ou snapshot de cluster foi inventado. Todos os artefatos em `docs/evidence/2026-11-10/` foram gerados pela execução real dos 6 containers e pods no ambiente local.
2. **Distinção Clara entre Local e Nuvem**: Manifestos de preparação para AWS EKS/RDS e `Jenkinsfile` estão versionados e validados sintaticamente, mas são explicitamente declarados como **não executados em conta AWS paga** durante esta etapa local.
3. **Responsabilidade na Defesa Individual**: O integrante é responsável por explicar na arguição oral o funcionamento detalhado do código, validação JWT, Cache-Aside e invalidação no Redis, publicação/consumo e idempotência no RabbitMQ, conversão cambial em Python, HPA e configuração Kubernetes.
