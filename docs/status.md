# Estado atual — 12/11/2026

Implementação do handoff de B: Secrets isolados por serviço, DNS reduzido em Product/Order, ACK após persistência, publisher confirms, requeue e controle global com opt-in. Exchange mantém seu código de integração; apenas o estágio Test foi alterado.

Os SHAs e testes desta rodada estão no [registro do handoff](handoff-2026-11-12.md): 9 testes unitários, smoke, 40 verificações integradas e resiliência com Redis/RabbitMQ reais passaram em Compose isolado. Compilação no Jenkins, Kubernetes com as imagens novas e EKS/RDS/TLS permanecem pendentes. Os relatórios abaixo são históricos de 10/11/2026, preservados sem reexecução nesta rodada; não comprovam os commits novos nem garantia de exatamente uma vez.

---

# Estado das Entregas e Evidências Medicas (Pessoa B)

Registro consolidado em **10/11/2026** por **Gustavo Lima Affonso Ferreira (Pessoa B)**. Todas as medições abaixo foram executadas contra os **6 microsserviços reais integrados** (`account`, `auth`, `gateway`, `product`, `order`, `exchange`) mais `PostgreSQL 16`, `Redis 7`, `RabbitMQ 3.13`, `Prometheus` e `Grafana`.

## 1. Matriz de Estado das Entregas

| Entrega | Responsável | Estado Atual | Evidência / Verificação |
|---|---|---|---|
| `store-product` (Python/FastAPI + Cache Redis) | Pessoa B | **Concluído e Publicado** | Testes unitários (`pytest`), Compose e Kubernetes (`7daad6f`) |
| `store-exchange` (Python/FastAPI Obrigatório + Fallback FX) | Pessoa B | **Concluído e Publicado** | Testes unitários (`pytest`), Compose e Kubernetes (`e353943`) |
| `store-order` (Python/FastAPI + Outbox/RabbitMQ Idempotente) | Pessoa B | **Concluído e Publicado** | Testes unitários (`pytest`), Compose e Kubernetes (`d0a05a5`) |
| Integração dos 6 serviços no `store-root` (Submódulos, Compose, K8s) | Dupla (A + B) | **Concluído e Validado** | `tests/smoke.py` e `tests/persistence.py` passaram em Compose e K8s |
| Gargalo B1: Cache Redis (`Cache-Aside`, TTL, Invalidação) | Pessoa B | **Comprovado com Medições** | `2,33x` mais vazão e `-61,1%` de latência `p95` sob carga em Compose; `1,48x–1,56x` em K8s |
| Gargalo B2: Mensageria RabbitMQ (Desacoplamento + Idempotência) | Pessoa B | **Comprovado com Medições** | 12 pedidos criados com consumidor pausado (`avg 23,3 ms`), drenagem em `237 ms` e `0` duplicatas após replay |
| Cluster Kubernetes Local (`kind-microserv-store`) com 6 serviços + HPA | Dupla (A + B) | **Concluído e Validado** | 11 Deployments saudáveis; `product`, `order` e `gateway` escalaram de `2` para `5` réplicas sob carga |
| Revisão Cruzada de B sobre A (`Account`, `Auth`, `Gateway`) | Pessoa B | **Concluída** | Documentada em [Testes de carga e revisão cruzada](load-testing.md) |
| Publicação da Documentação (`store-docs-b` via GitHub Pages) | Pessoa B | **Concluída** | Compilação `mkdocs build --strict` e workflow GitHub Pages ativo |
| Deploy em nuvem paga AWS (EKS / RDS) e Jenkins remoto | Dupla (A + B) | **Preparado (Não Executado)** | Manifestos `eksctl.yaml`, `rds.yaml` e `Jenkinsfile` prontos; não executados em conta paga |

---

## 2. Resultados do Gargalo B1 — Cache Redis no Product Service

### 2.1 Benchmark Controlado (`tests/cache_bottleneck.py` — 8 usuários concorrentes, 240 requisições/fase)

| Ambiente | Operação | Modo | Vazão (`req/s`) | Latência `p50` | Latência `p95` | Ganho de Vazão | Redução de `p95` |
|---|---|---|---:|---:|---:|---:|---:|
| **Docker Compose** | `GET /products/{id}` | Sem Cache (`BYPASS`) | `342,57` | `22,08 ms` | `35,01 ms` | — | — |
| **Docker Compose** | `GET /products/{id}` | **Com Cache Redis (`HIT`)** | **`676,07`** | **`10,44 ms`** | **`16,83 ms`** | **`1,97x`** | **`-51,94%`** |
| **Docker Compose** | `GET /products` (lista) | Sem Cache (`BYPASS`) | `300,51` | `26,24 ms` | `36,13 ms` | — | — |
| **Docker Compose** | `GET /products` (lista) | **Com Cache Redis (`HIT`)** | **`655,38`** | **`11,60 ms`** | **`18,41 ms`** | **`2,18x`** | **`-49,05%`** |
| **Kubernetes (`kind`)** | `GET /products/{id}` | Sem Cache (`BYPASS`) | `542,29` | `14,11 ms` | `20,43 ms` | — | — |
| **Kubernetes (`kind`)** | `GET /products/{id}` | **Com Cache Redis (`HIT`)** | **`801,71`** | **`9,67 ms`** | **`13,36 ms`** | **`1,48x`** | **`-34,60%`** |
| **Kubernetes (`kind`)** | `GET /products` (lista) | Sem Cache (`BYPASS`) | `508,58` | `14,63 ms` | `20,13 ms` | — | — |
| **Kubernetes (`kind`)** | `GET /products` (lista) | **Com Cache Redis (`HIT`)** | **`793,82`** | **`9,77 ms`** | **`14,14 ms`** | **`1,56x`** | **`-29,77%`** |

Verificações funcionais automatizadas em ambos os ambientes:
- Primeira leitura após escrita: `X-Cache: MISS` (popula Redis);
- Segunda leitura: `X-Cache: HIT`;
- Remoção (`DELETE /products/{id}`): status `204` com invalidação imediata da chave e leitura subsequente retornando `404`.

---

## 3. Resultados do Gargalo B2 — Mensageria RabbitMQ no Order Service

Execução de `tests/messaging_bottleneck.py` em Docker Compose e Kubernetes:

| Indicador Verificado | Docker Compose | Kubernetes (`kind`) | Significado Técnico |
|---|---:|---:|---|
| Pedidos criados com consumidor pausado (`POST /orders`) | `12` (`100% HTTP 201`) | `12` (`100% HTTP 201`) | API de checkout permanece rápida e disponível mesmo com worker de notificação parado |
| Latência média do `POST /orders` durante pausa | `23,34 ms` | `29,78 ms` | Publicação na fila não bloqueia o cliente pelo processamento downstream |
| Status da notificação durante a pausa | `PENDING` | `PENDING` | Eventos retidos com segurança na fila durável `order.events` |
| Pedidos processados após reativar consumidor | `12 / 12` | `12 / 12` | Zero perda de mensagens (*at-least-once delivery*) |
| Tempo de *catch-up* para drenar o lote | `237,74 ms` | `389,83 ms` | Recuperação rápida do backlog acumulado |
| Notificações únicas após 2 replays de evento duplicado | `12` (1 por pedido) | `12` (1 por pedido) | **Idempotência comprovada**: `duplicate_deliveries_skipped >= 2`, sem duplicar registros |

---

## 4. Testes de Carga Sustentada (`scripts/load.py`) e Escalabilidade HPA

| Ambiente | Cenário | Usuários | Duração | Requisições | Vazão (`req/s`) | `p50` | `p95` | `p99` | Erros |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|
| **Compose** | `products_cached` (`HIT`) | 8 | 15s | `13.358` | **`890,15`** | `8,82 ms` | `11,71 ms` | `14,10 ms` | `0` |
| **Compose** | `products_uncached` (`BYPASS`) | 8 | 15s | `5.743` | **`382,58`** | `20,18 ms` | `30,13 ms` | `37,81 ms` | `0` |
| **Compose** | `orders` (`POST /orders`) | 8 | 15s | `341` | `22,50` | `350,41 ms` | `490,65 ms` | `553,40 ms` | `0` |
| **Compose** | `business_mix` (6 serviços) | 8 | 15s | `1.664` | `109,46` | `2,59 ms` | `409,74 ms` | `478,93 ms` | `0` |
| **Kubernetes** | `products_cached` (`HIT`, 2 pods) | 8 | 15s | `11.051` | **`736,35`** | `10,52 ms` | `15,93 ms` | `19,93 ms` | `0` |
| **Kubernetes** | `products_uncached` (`BYPASS`, 2 pods) | 8 | 15s | `9.219` | **`614,18`** | `12,80 ms` | `17,65 ms` | `21,61 ms` | `0` |
| **Kubernetes** | `business_mix` + **HPA (12 usuários)** | 12 | 45s | `4.745` | **`105,15`** | `5,58 ms` | `667,11 ms` | `911,13 ms` | `0` |

### Distribuição entre Réplicas e Escalonamento Horizontal (HPA) no Kubernetes

- Na rodada `products_cached` em Kubernetes, o Service `product` balanceou quase perfeitamente as `11.051` respostas entre os pods iniciais: `product-67464496f8-rqvvv` (`5.547`) e `product-67464496f8-6km4t` (`5.504`).
- Durante a carga combinada `business_mix` com 12 usuários concorrentes (`k8s-business-mix.json` e `k8s-recorded-hpa-b.json`), o **HPA escalou automaticamente `product` de 2 para 5 pods (`5/5 READY`), `order` de 2 para 5 pods (`5/5 READY`) e `gateway` de 2 para 5 pods (`5/5 READY`)**.
- As `4.745` requisições da rodada `business_mix` foram distribuídas entre **5 réplicas de `product`** (`1.183`, `807`, `666`, `182` e `3`), **2 réplicas de `order`** (`486` e `467`) e **2 réplicas de `exchange`** (`536` e `415`), com **zero erros (`error_ratio: 0.0`)**.

---

## 5. Arquivos de Evidência Verificáveis (`docs/evidence/2026-11-10/`)

### Gargalos B1 (Redis) e B2 (RabbitMQ)
- [Benchmark Cache Redis — Docker Compose](evidence/2026-11-10/compose-cache-bottleneck.json)
- [Benchmark Cache Redis — Kubernetes](evidence/2026-11-10/k8s-cache-bottleneck.json)
- [Prova de Mensageria e Idempotência RabbitMQ — Docker Compose](evidence/2026-11-10/compose-messaging-bottleneck.json)
- [Prova de Mensageria e Idempotência RabbitMQ — Kubernetes](evidence/2026-11-10/k8s-messaging-bottleneck.json)

### Relatórios de Carga (`scripts/load.py`)
- [Carga `products_cached` — Compose](evidence/2026-11-10/compose-load-products-cached.json)
- [Carga `products_uncached` — Compose](evidence/2026-11-10/compose-load-products-uncached.json)
- [Carga `orders` — Compose](evidence/2026-11-10/compose-load-orders.json)
- [Carga `business_mix` — Compose](evidence/2026-11-10/compose-load-business-mix.json)
- [Carga `products_cached` — Kubernetes](evidence/2026-11-10/k8s-products-cached.json)
- [Carga `products_uncached` — Kubernetes](evidence/2026-11-10/k8s-products-uncached.json)
- [Carga `business_mix` — Kubernetes (durante HPA)](evidence/2026-11-10/k8s-business-mix.json)

### Snapshots de Cluster Kubernetes, HPA, Prometheus e Vídeo Técnico
- [Snapshots de HPA e `kubectl top pods` em tempo real](evidence/2026-11-10/k8s-recorded-hpa-b.json)
- [Vídeo técnico local de HPA e carga dos 6 serviços (`hpa-b-local.mp4`)](evidence/2026-11-10/hpa-b-local.mp4): replay renderizado a partir das saídas reais de `kubectl get hpa`, `kubectl get deployments` e `kubectl top pods` durante a carga concorrente no cluster `kind-microserv-store`.
- [Estado de Deployments, Pods, Services, HPAs, PDBs e `top pods`](evidence/2026-11-10/k8s-resources.txt)
- [Informações do Cluster Kind](evidence/2026-11-10/k8s-cluster.txt) e [Eventos do Namespace `store`](evidence/2026-11-10/k8s-events.txt)
- [Alvos Prometheus Ativos — Compose](evidence/2026-11-10/compose-prometheus-targets.json) e [Alvos Prometheus Ativos — Kubernetes](evidence/2026-11-10/k8s-prometheus-targets.json)
- [IDs Imutáveis das 6 Imagens Docker Locais](evidence/2026-11-10/k8s-images.json)
