# Execução, Kubernetes e Jenkins

## Ambiente local integrado

Na versão publicada de `store-root`, A deve atualizar os ponteiros somente para os commits publicados por B, registrados no [handoff](handoff-2026-11-12.md). Não recriar contratos de A nem reutilizar commits retirados pelo rollback.

```bash
git submodule update --init --recursive
python scripts/setup-local.py
docker compose -f compose.yaml -f compose.business.yaml up --build -d
python tests/smoke.py
python tests/business_integration.py
python tests/business_resilience.py
```

O teste de resiliência interrompe/reinicia Redis e RabbitMQ somente no projeto Compose local isolado. Não executar contra ambiente compartilhado. O aceite usa consulta externa real de câmbio; doubles pertencem apenas aos testes unitários. Não apagar volumes para resolver inicialização do RabbitMQ.

## Kubernetes e Secrets

Namespace `store`; Services internos `product:8000`, `order:8000`, `exchange:8000`, `redis:6379` e `rabbitmq:5672`. Somente Gateway expõe a API externa.

| Serviço | Variável | Chave em `store-secrets` |
|---|---|---|
| Product | `DATABASE_URL` | `PRODUCT_DATABASE_URL` |
| Product | `REDIS_URL` | `REDIS_URL` |
| Order | `DATABASE_URL` | `ORDER_DATABASE_URL` |
| Order | `RABBITMQ_URL` | `RABBITMQ_URL` |
| Product, Order, Exchange | `JWT_SECRET` | `JWT_SECRET` |

URLs autenticadas ficam em Secret, fora de ConfigMap e Git. Product e Order usam bancos/usuários próprios. `scripts/k8s-secrets.py --business --local-database` prepara o ambiente local; em AWS fornecer URLs reais. Preservar o `.env` privado; setup acrescenta chaves ausentes sem substituir segredos existentes.

Product/Order usam `dnsConfig.options` com `timeout=1` e `attempts=1` para reduzir espera antes do fallback/outbox. Revalidar budgets no EKS. `ENABLE_DEMO_CONTROLS: "false"` permanece no ConfigMap de Order; habilitar apenas para demonstração controlada.

Após construir/carregar as seis imagens novas, na raiz:

```powershell
./scripts/deploy-local.ps1 -Context kind-microserv-store -Business
kubectl --context kind-microserv-store -n store port-forward service/gateway 8082:8000
```

Em outro terminal:

```powershell
$env:BASE_URL='http://127.0.0.1:8082'
python tests/business_integration.py
```

## Jenkins por serviço

Sequência preservada: Checkout → Test → Build → Publish → Deploy. Agente Linux `docker-kubectl` com Docker funcional e kubectl. O estágio Test usa `python:3.12-slim`, checkout somente leitura (`/src:ro`), `PYTHONPATH=/src`, diretório `/tmp` e `pytest /src/tests -q -p no:cacheprovider`; falha interrompe os próximos estágios. SQLite temporário não suja o checkout.

Configurar SCM privado com credencial de leitura, `DOCKERHUB_NAMESPACE` válido, credencial `dockerhub` e `store-kubeconfig`. Imagens usam `store-<serviço>:<SHA completo>`. A primeira validação mantém `DEPLOY=false`. Somente após escolher/configurar o cluster habilitar deploy, com `kubectl apply --dry-run=server` e espera por rollout.

A presença dos Jenkinsfiles não prova funcionamento do pipeline. Ainda validar compilação real, checkout privado em agente limpo, falha deliberada bloqueando publicação/deploy, publicação SHA, fluxo Gateway e rollback. O job integrado da raiz usa `github-read`, Git tool `Default`, porta 18080 e seis serviços; execução real permanece pendente.

```bash
kubectl -n store set image deployment/product product=<namespace>/store-product:<sha-anterior>
kubectl -n store rollout status deployment/product --timeout=180s
```

## AWS e limites

EKS, RDS real, TLS público, custos completos e execução remota Jenkins ainda dependem de validação no destino. Não foram provisionados recursos pagos nesta rodada. Consultar [estado e evidências](status.md) para separar medições históricas da implementação atual.
