# Custos, PaaS, SLA e Impacto dos Serviços da Pessoa B

## 1. Extensão da Estimativa de Custos para a Topologia Completa (A + B)

Na arquitetura completa com os 6 microsserviços (`account`, `auth`, `gateway`, `product`, `order`, `exchange`) mais cache (`Redis`) e mensageria (`RabbitMQ`), a análise de custos na AWS (`us-east-1`) distingue claramente duas opções de implantação para as dependências da Pessoa B:

| Componente | Opção 1: Auto-hospedado no EKS (Pods no Cluster) | Opção 2: Serviços Gerenciados AWS (PaaS) | Impacto Técnico e Financeiro |
|---|---|---|---|
| **Control Plane Kubernetes** | Amazon EKS Standard (`US$ 0,10/hora` = `US$ 73,00/mês` em 730h) | Mesmo | Custo fixo do plano de controle (verificado na página oficial de preços do EKS) |
| **Workers Kubernetes (6 serviços × 2 pods mín = 12 pods de API)** | 2 a 3 nós `t3.medium` (2 vCPU, 4 GiB RAM) com volumes EBS `gp3` de 20 GiB | 2 nós `t3.medium` | Com 12 pods de aplicação + Prometheus/Grafana, `t3.small` ficaria saturado em memória; `t3.medium` acomoda os requests de CPU/memória |
| **Banco Relacional (`Account`, `Product`, `Order`)** | Pod PostgreSQL local (apenas QA/Kind) | Amazon RDS for PostgreSQL `db.t4g.micro` Single-AZ (20 GiB `gp3`) | Isola persistência dos ciclos de vida dos pods; Single-AZ reduz custo acadêmico, mas tem SLA inferior ao Multi-AZ |
| **Cache de Produtos (`Redis`)** | Pod `redis:7.4-alpine` no namespace `store` (custo marginal zero além dos nós EC2) | Amazon ElastiCache for Redis (`cache.t4g.micro`) | Em janela acadêmica curta, rodar Redis no próprio cluster economiza uma instância dedicada; em produção, ElastiCache oferece failover gerenciado |
| **Broker de Eventos (`RabbitMQ`)** | Pod `rabbitmq:3.13-alpine` no namespace `store` com PVC `gp3` | Amazon MQ for RabbitMQ (`mq.t3.micro`) | Em ambiente acadêmico, pod no cluster evita custo fixo horário do Amazon MQ; em produção, Amazon MQ reduz carga operacional de manutenção do broker |

> [!WARNING]
> **Distinção entre Janela Acadêmica e Operação Mensal (730h)**: Em uma janela de demonstração acadêmica de 48h a 72h, o custo do plano de controle EKS é de `US$ 4,80` a `US$ 7,20` (sem contar EC2, EBS, RDS, NAT Gateway e Load Balancer). Todos os recursos devem ser removidos imediatamente após a avaliação seguindo o checklist de teardown em [Execução e deploy](deploy.md) para evitar cobrança contínua de NAT Gateway, LoadBalancer, instâncias RDS e snapshots retidos.

---

## 2. Enquadramento PaaS vs. Responsabilidade da Equipe

- **Amazon EKS**: gerencia apenas o *control plane* Kubernetes (API server, `etcd`, controladores). A equipe continua integralmente responsável pelos `Deployments`, `Services`, `HPA`, `PDB`, imagens Docker, segredos, `NetworkPolicies` e dimensionamento dos nós workers.
- **Amazon RDS / ElastiCache / Amazon MQ**: automatizam provisionamento de máquina, patches de sistema operacional e backups básicos, mas **não resolvem gargalos de aplicação**:
  - Sem a lógica **Cache-Aside e invalidação** implementada no código do `store-product`, um Redis gerenciado não evita leituras repetidas no banco nem previne dados obsoletos.
  - Sem o **Transactional Outbox e a chave de idempotência (`UNIQUE (order_id)`)** implementados no código do `store-order`, um broker RabbitMQ gerenciado não impede duplicação de efeitos colaterais em reentregas (*at-least-once delivery*).

---

## 3. Metas da Aplicação (SLO) vs. SLA de Fornecedor

| Métrica / Indicador | Alvo da Aplicação (SLO em Carga Local Estável) | Resultado Medido (Local) | Observação Técnica |
|---|---|---|---|
| Latência `p95` de leitura de produto com Cache Redis (`GET /products/{id}`) | `< 50 ms` com 8 usuários concorrentes | Ver [Estado e evidências](status.md) | Ganho direto da eliminação de round-trip SQL em `X-Cache: HIT` |
| Latência `p95` de criação de pedido (`POST /orders`) | `< 250 ms` com 8 usuários concorrentes | Ver [Estado e evidências](status.md) | Desacoplamento via RabbitMQ evita bloquear o checkout pelo processamento de notificação |
| Taxa de erros `5xx` em carga nominal | `< 1%` | `0%` nos testes locais | Requer dependências saudáveis (`postgres`, `redis`, `rabbitmq`) |
| Disponibilidade mensal contratual AWS EKS / RDS | N/A para ambiente local | SLA EKS Standard: `99,95%` (control plane); SLA RDS Single-AZ: créditos apenas abaixo de `99,5%` | SLA de infraestrutura da AWS **não garante** disponibilidade dos pods da aplicação |
