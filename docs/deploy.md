# Execução, Kubernetes, CI/CD Jenkins e Preparação AWS

## 1. Execução Completa via Docker Compose (6 Serviços + Monitoramento)

Na raiz do repositório integrador (`Microserv-1/store-root`):

```bash
git clone --recurse-submodules https://github.com/Microserv-1/store-root.git
cd store-root
python3 scripts/setup-local.py
docker compose --profile monitoring up --build -d
```

Verificação automatizada ponta a ponta dos 6 serviços, cache Redis e mensageria RabbitMQ:

```bash
python3 tests/smoke.py
python3 tests/persistence.py
python3 tests/cache_bottleneck.py
python3 tests/messaging_bottleneck.py
```

Portas locais expostas (vinculadas a `127.0.0.1`):
- **Gateway (única entrada de API)**: `http://127.0.0.1:8080`
- **Prometheus**: `http://127.0.0.1:9090`
- **Grafana**: `http://127.0.0.1:3000` (`admin` / senha em `GRAFANA_PASSWORD` no `.env`)

---

## 2. Cluster Kubernetes Local (`kind` / `minikube`)

Todos os 6 serviços (`account`, `auth`, `gateway`, `product`, `order`, `exchange`) rodam simultaneamente no namespace `store`, acompanhados por `postgres`, `redis`, `rabbitmq`, `prometheus`, `grafana` e `metrics-server`.

```bash
# 1. Criar cluster Kind
kind create cluster --config infra/k8s/kind.yaml

# 2. Carregar imagens locais dos 6 serviços no cluster
kind load docker-image \
  microserv-store/account:local \
  microserv-store/auth:local \
  microserv-store/gateway:local \
  microserv-store/product:local \
  microserv-store/order:local \
  microserv-store/exchange:local \
  --name microserv-store

# 3. Aplicar secrets, dependências (Postgres, Redis, RabbitMQ), 6 Deployments/Services/HPAs/PDBs e monitoramento
python3 scripts/deploy-local.py --context kind-microserv-store

# 4. Expor o Service do Gateway para execução dos testes de aceite e carga
kubectl --context kind-microserv-store -n store port-forward service/gateway 8081:8000
```

Em outro terminal:

```bash
BASE_URL=http://127.0.0.1:8081 python3 tests/smoke.py
BASE_URL=http://127.0.0.1:8081 python3 tests/cache_bottleneck.py --output artifacts/k8s-cache-bottleneck.json
BASE_URL=http://127.0.0.1:8081 python3 tests/messaging_bottleneck.py --output artifacts/k8s-messaging-bottleneck.json
```

### Especificação Kubernetes dos Serviços da Pessoa B

Cada repositório (`store-product`, `store-order`, `store-exchange`) possui `k8s/k8s.yaml` contendo:
- `ConfigMap` com URLs internas e parâmetros de cache/fila;
- `Deployment` com `replicas: 2`, `strategy: RollingUpdate` (`maxUnavailable: 0`, `maxSurge: 1`), `securityContext` não-root (`runAsUser: 10001`), `readinessProbe` (`/ready`) e `livenessProbe` (`/health-check`), `resources.requests` (`cpu: 100m`, `memory: 128Mi`) e `limits` (`cpu: 500m`, `memory: 256Mi`);
- `Service` `ClusterIP` na porta `8000`;
- `HorizontalPodAutoscaler` (`autoscaling/v2`) escalando de `minReplicas: 2` até `maxReplicas: 5` com meta de `60%` de utilização média de CPU;
- `PodDisruptionBudget` com `minAvailable: 1`.

---

## 3. Pipelines CI/CD (Jenkinsfile por Serviço)

Cada um dos 3 repositórios da Pessoa B (`store-product`, `store-order`, `store-exchange`) possui seu próprio `Jenkinsfile` declarativo com 4 estágios:

1. **Test**: Cria virtualenv Python, instala `requirements.txt` e executa `python -m pytest -q`. Qualquer falha aborta o pipeline antes do build da imagem.
2. **Build Image**: Resolve o `GIT_SHA` completo (`git rev-parse HEAD`) e constrói a imagem Docker imutável `<DOCKERHUB_NAMESPACE>/store-<service>:<GIT_SHA>`.
3. **Push Image**: Autentica no Docker Hub via credencial `dockerhub` e publica apenas a tag imutável do commit (`:<GIT_SHA>`), evitando tags mutáveis como `:latest`.
4. **Deploy to Kubernetes** (quando `params.DEPLOY == true`): Substitui o placeholder de imagem em `k8s/k8s.yaml` pela imagem `<DOCKERHUB_NAMESPACE>/store-<service>:<GIT_SHA>`, executa validação server-side (`kubectl apply --dry-run=server`), aplica no namespace `store` e aguarda `kubectl rollout status deployment/<service> --timeout=180s`.

### Procedimento de Rollback Auditável

Em caso de regressão detectada após deploy:
```bash
kubectl -n store set image deployment/product product=<namespace>/store-product:<git-sha-anterior>
kubectl -n store rollout status deployment/product --timeout=180s
```

---

## 4. Preparação para AWS (EKS + RDS) e Teardown

Os manifestos de preparação AWS estão em `store-root/infra/aws/` (`eksctl.yaml` e `rds.yaml`), com suporte a renderização de imagens imutáveis via `python3 scripts/render-deploy.py`.

> [!NOTE]
> Conforme diretriz de transparência técnica do projeto, **nenhum recurso pago na AWS foi provisionado durante a validação local**. Os testes funcionais, de gargalos (Redis e RabbitMQ), de carga e de HPA foram integralmente executados e comprovados em **Docker Compose** e no cluster **Kubernetes local (`kind-microserv-store`)**.
