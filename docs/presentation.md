# Apresentação em Dupla e Guia de Defesa Individual (Pessoa B)

## 1. Roteiro Integrado da Apresentação em Vídeo (1 a 2 minutos — Pessoa A + Pessoa B)

O vídeo de apresentação final (1–2 minutos) deve contar com a participação oral de **Yuri Henrique (Pessoa A)** e **Gustavo Lima Affonso Ferreira (Pessoa B)**, separado do vídeo técnico de terminal/HPA:

| Intervalo | Integrante | Conteúdo e Demonstração |
|---|---|---|
| **0:00 – 0:15** | Ambos (A + B) | Apresentação da dupla, organização `Microserv-1` e visão geral dos 6 microsserviços integrados atrás do Gateway Go |
| **0:15 – 0:40** | Pessoa A (Yuri) | Fluxo de identidade e segurança: `Account` (Python/Argon2id), `Auth` (Java/Spring Boot JWT HS256), `Gateway` (Go, anti-spoofing de `X-Account-ID`), observabilidade (Prometheus/Grafana) e balanceamento/HPA |
| **0:40 – 1:10** | **Pessoa B (Gustavo)** | Demonstração funcional de negócio: criação e busca de produtos (`store-product`), cotação de moedas em Python (`store-exchange`), criação de pedido em USD e consulta convertida para BRL (`store-order`) com isolamento entre contas |
| **1:10 – 1:40** | **Pessoa B (Gustavo)** | Demonstração dos 2 gargalos da Pessoa B: **(B1)** comparação medida de latência/vazão com Cache Redis (`X-Cache: HIT` vs `BYPASS` e invalidação em `DELETE`); **(B2)** resiliência assíncrona com RabbitMQ (criação de pedidos com consumidor pausado, retomada e prova de idempotência via replay) |
| **1:40 – 2:00** | Ambos (A + B) | Síntese dos resultados em Kubernetes (`kind-microserv-store` com os 6 serviços e HPA), análise de custos/SLA e encerramento |

---

## 2. Pontos-Chave para a Defesa Individual da Pessoa B

Resumo rápido das perguntas de arguição sobre o código da Pessoa B:

1. **Como o `Order Service` garante que um usuário não acesse o pedido de outro?**
   - Tanto `Order` quanto `Gateway` validam o JWT HS256. O `Order Service` extrai `account_id = payload["sub"]` do token e filtra `WHERE id = :order_id AND id_account = :account_id`. Se pertencer a outra conta, retorna `404 Not Found`.
2. **Como funciona a conversão de moeda em `GET /orders/{id}?currency=BRL`?**
   - O pedido é sempre persistido em `USD` no PostgreSQL. Quando `currency != USD`, o `Order Service` chama `GET /exchanges/USD/{currency}` repassando o header `Authorization: Bearer <token>`, obtém o campo `sell` e multiplica os preços unitários, subtotais e total em memória com `Decimal.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)`, sem alterar o registro no banco.
3. **Como o Cache Redis evita retornar um produto que acabou de ser deletado?**
   - Em `DELETE /products/{id}`, após remover a linha no PostgreSQL, o serviço executa `redis.delete(f"product:item:{id}")` e remove todas as chaves `product:list:*` antes de responder `204 No Content`.
4. **O que acontece se o RabbitMQ entregar duas vezes o mesmo evento `order.created`?**
   - A tabela `order_notifications` possui constraint `UNIQUE (order_id)`. O consumidor verifica se já existe notificação para aquele `order_id`; se existir, incrementa apenas `delivery_attempts` e envia `basic_ack`, garantindo idempotência.
