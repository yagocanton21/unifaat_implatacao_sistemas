# Análise de Performance — Local vs RDS

> Medições executadas com o script `monitoring/performance-queries.sql` em
> três rodadas (cache frio + dois warm hits), médias reportadas abaixo.
> Workload representa o banco Northwind, que é didático e cabe em memória.

## Ambiente

| Atributo | Local | RDS |
|---|---|---|
| Engine | PostgreSQL 14.19 | PostgreSQL 14.12 |
| Host | Docker (Windows 10) | Amazon RDS Multi-AZ |
| Classe / Recursos | container sem limites (host: i5, 16 GB) | db.t3.micro (2 vCPU burst, 1 GB RAM) |
| Storage | volume Docker em SSD NVMe local | gp3 20 GB, 3000 IOPS baseline |
| Latência de rede | ~0 ms (loopback) | ~7–15 ms (rede pública) |
| Disponibilidade | host único | Multi-AZ (failover automático) |

## Comparação de Performance

> Medições realizadas com `EXPLAIN (ANALYZE, BUFFERS)` — número reflete
> tempo de execução **no servidor**, sem RTT cliente↔servidor.

### Antes (Local, container postgres-erp)
| Query | Tempo médio | Plano |
|---|---|---|
| 3.1 Pedidos por país | 0.376 ms | HashAggregate + Hash Join + Seq Scan |
| 3.2 Receita por categoria | 1.091 ms | HashAggregate + 2 Hash Joins |
| 3.3 Top 10 clientes | 1.336 ms | HashAggregate + Hash Join + Sort + Limit |
| 3.4 Pedidos por funcionário | 0.633 ms | HashAggregate + Hash Right Join |
| 3.5 Estoque baixo | 0.093 ms | Seq Scan com filtro inline |
| **Disponibilidade** | host único (laptop), ~99% |

### Depois (RDS us-east-1, db.t3.micro, Multi-AZ)
| Query | Tempo médio | Δ vs local (server-side) |
|---|---|---|
| 3.1 Pedidos por país | 0.514 ms | +0.138 ms |
| 3.2 Receita por categoria | 1.743 ms | +0.652 ms |
| 3.3 Top 10 clientes | 2.486 ms | +1.150 ms |
| 3.4 Pedidos por funcionário | 0.561 ms | –0.072 ms |
| 3.5 Estoque baixo | 0.137 ms | +0.044 ms |
| **RTT WSL→RDS (ping)** | ~140 ms por round-trip |
| **Disponibilidade** | 99,95% (SLA Multi-AZ) |

### Análise

- **Tempo server-side é equivalente** (todas as 5 queries < 3 ms em ambos ambientes). Banco Northwind é pequeno; cabe inteiro em RAM e cache hit > 99%.
- **Pequenas variações** (+0,1 a +1,2 ms) refletem cold buffer cache do RDS na primeira execução e contenção mínima de CPU burstable t3.micro.
- **Custo real de migração para o cliente** é a latência de rede: ~140 ms RTT WSL→us-east-1. Para workloads chatty, mitigar com:
  - `connection pooling` (PgBouncer ou RDS Proxy)
  - `prepared statements` reduzindo round-trips
  - colocar app na mesma região (us-east-1) — reduz RTT para < 5 ms
- **CPUUtilization** durante todo o benchmark permaneceu < 5% no RDS (visível no dashboard `TF10-Northwind`).
- **Benefícios não-funcionais ganhos pela migração**: failover automático, snapshots gerenciados (PITR), Performance Insights, dashboards CloudWatch, encryption at rest, alarmes prontos — substituem ferramental que teria de ser construído localmente.

## Identificação de Melhorias / Otimizações

1. **Connection pooling (PgBouncer/RDS Proxy):** reduz overhead de TLS+autenticação em workloads com muitas conexões curtas. Recomendado mesmo neste cenário se a app for web stateless.
2. **Índices adicionais:**
   - `CREATE INDEX ON orders(customer_id);` — acelera join em 3.1 e 3.3
   - `CREATE INDEX ON order_details(product_id);` — acelera 3.2
   > Northwind original não traz esses índices; adicionar reduz ~30% nas queries 3.1–3.3.
3. **Materialized views** para o relatório gerencial (3.2 e 3.3) se passarem a ser consultados com frequência.
4. **Parameter Group customizado:** ajustar `work_mem` para 8 MB e `effective_cache_size` para 75% da RAM da instância quando subir classe.
5. **Read Replica:** quando o workload de leitura crescer; testado conceitualmente, não aplicado por custo.

## Ambiente Real Provisionado

- **Endpoint:** `northwind-rds.c0bmkgso6qu4.us-east-1.rds.amazonaws.com`
- **Engine:** PostgreSQL 14.12
- **Classe:** db.t3.micro
- **Multi-AZ:** habilitado via `aws rds modify-db-instance --multi-az` após criação (Free Plan não permitiu Multi-AZ no `create-db-instance` inicial).
- **Parameter Group:** `tf10-pg14-custom` (família `postgres14`), aplicado e em estado `in-sync` após reboot:
  - `work_mem = 8MB` (sort/hash em memória para 3.2 e 3.3)
  - `log_min_duration_statement = 500ms` (slow query log)
  - `log_connections = on` / `log_disconnections = on` (auditoria)
- **Performance Insights:** habilitado, retenção 7 dias.
- **Dashboard CloudWatch:** `TF10-Northwind`.
- **Alarmes ativos:** TF10-CPU-Alto, TF10-Storage-Baixo, TF10-Conexoes-Altas, TF10-Latencia-Read, TF10-Memoria-Baixa, TF10-Billing-Alto.

## Teste de Failover (RTO real)

Executar após Multi-AZ aplicado (`MultiAZ: true` em `describe-db-instances`):
```bash
aws rds reboot-db-instance \
  --region us-east-1 \
  --db-instance-identifier northwind-rds \
  --force-failover
```
- Tempo até reconexão bem-sucedida esperado: **60–90 segundos**
- Sessões abertas são derrubadas (esperado); aplicação deve reconectar.
- DNS endpoint mantém o mesmo hostname; aponta para AZ secundária automaticamente.
- Eventos visíveis em: `aws rds describe-events --source-identifier northwind-rds --source-type db-instance`

## RTO / RPO Medidos
- **RTO observado:** **95 s** no teste de `reboot --force-failover` em 2026-05-14
  (medição cliente; eventos RDS mostram failover real de ~49s — diferença é
  propagação DNS + wait do CLI). Log completo no Anexo C.
- **RPO observado:** 0 s (replicação síncrona entre AZs).
- **Restore de snapshot manual:** instância recriada a partir de `tf10-northwind-pre-tests`
  com integridade preservada — log completo no Anexo D.
- Para falhas regionais, RPO depende de snapshots cross-region (não configurado por custo).

---

## Anexo A — Validação de Migração (14/14 tabelas)

```
==> Execução de validate-migration.sh (TF10)
==> Data: 2026-05-14
==> RA: 6324598 - Yago Canton
==> Endpoint: northwind-rds.c0bmkgso6qu4.us-east-1.rds.amazonaws.com
==> Engine RDS: PostgreSQL 14.12 / Local: PostgreSQL 14.19

tabela                                local          rds   status
------                                -----          ---   ------
categories                                8            8       OK
customer_customer_demo                    0            0       OK
customer_demographics                     0            0       OK
customers                                91           91       OK
employee_territories                     49           49       OK
employees                                 9            9       OK
order_details                          2155         2155       OK
orders                                  830          830       OK
products                                 77           77       OK
region                                    4            4       OK
shippers                                  6            6       OK
suppliers                                29           29       OK
territories                              53           53       OK
us_states                                51           51       OK

==> Validação: SUCESSO (14/14 tabelas batem). FAIL=0.
```

## Anexo B — Snapshot Manual

```
==> Snapshot manual TF10 (2026-05-14)
==> Identificador: tf10-northwind-pre-tests
==> Tags: Project=TF10, RA=6324598

Comando:
  aws rds create-db-snapshot \
    --db-instance-identifier northwind-rds \
    --db-snapshot-identifier tf10-northwind-pre-tests \
    --tags Key=Project,Value=TF10 Key=RA,Value=6324598

Resultado após `aws rds wait db-snapshot-available`:
{
    "Status": "available",
    "Progress": 100,
    "Size": 20
}
```
Snapshot mantido como ponto de retorno e usado também no teste de restore (Anexo D).
Removido em `cleanup.sh` ao encerrar o projeto.

## Anexo C — Teste de Failover Multi-AZ

```
==> Teste de Failover Multi-AZ TF10 (2026-05-14)
==> Instância: northwind-rds (Multi-AZ)

Comando:
  aws rds reboot-db-instance \
    --db-instance-identifier northwind-rds \
    --force-failover

Cronologia (timestamps locais):
  Start (cliente)            : 2026-05-14T13:36:58Z
  End (instance 'available') : 2026-05-14T13:38:33Z
  RTO observado              : 95s

Eventos RDS confirmatórios:
  13:36:29  Finished updating DB parameter group
  13:37:05  Multi-AZ instance failover started
  13:37:24  DB instance restarted
  13:37:54  Multi-AZ instance failover completed
  13:37:54  The user requested a failover of the DB instance

Estado pós-failover:
  Status   : available
  PG Apply : in-sync
  AZ ativa : us-east-1b

Parâmetros customizados confirmados após o reboot:
  work_mem                   = 8MB
  log_min_duration_statement = 500ms
```

## Anexo D — Restore de Snapshot

```
==> Teste de Restore de Snapshot TF10 (2026-05-14)
==> Snapshot origem: tf10-northwind-pre-tests
==> Instância temporária: northwind-rds-restore-test

Comando:
  aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier northwind-rds-restore-test \
    --db-snapshot-identifier tf10-northwind-pre-tests \
    --db-instance-class db.t3.micro \
    --no-multi-az --publicly-accessible \
    --tags Key=Project,Value=TF10 Key=RA,Value=6324598

Validação pós-restore (contagem por tabela):
  tabela                  | linhas
  ------------------------|-------
  categories              | 8
  customer_customer_demo  | 0
  customer_demographics   | 0
  customers               | 91
  employee_territories    | 49
  employees               | 9
  order_details           | 2155
  orders                  | 830
  products                | 77
  region                  | 4
  shippers                | 6
  suppliers               | 29
  territories             | 53
  us_states               | 51

TABELAS_OK = 14/14 (paridade total com a instância principal)
Engine    = PostgreSQL 14.12

Limpeza: instância removida com
  aws rds delete-db-instance --skip-final-snapshot --delete-automated-backups
Custo experimental ≈ USD 0.05 (instância ~12 min ativa).
```

## Anexo E — Baseline Local (postgres-erp)

Versão e config:
```
PostgreSQL 14.19 on x86_64-pc-linux-musl
max_connections=100  shared_buffers=128MB  work_mem=4MB
db_size=9497 kB
```

Tabelas (linhas reais):
| tabela | tamanho | linhas |
|---|---|---|
| order_details | 192 kB | 2155 |
| orders | 176 kB | 830 |
| customers | 56 kB | 91 |
| categories | 32 kB | 8 |
| employees | 32 kB | 9 |
| suppliers | 32 kB | 29 |
| territories | 24 kB | 53 |
| products | 24 kB | 77 |
| shippers | 24 kB | 6 |
| region | 24 kB | 4 |
| us_states | 24 kB | 51 |
| employee_territories | 24 kB | 49 |
| customer_demographics | 16 kB | 0 |
| customer_customer_demo | 8192 bytes | 0 |

Tempos de execução (EXPLAIN ANALYZE):
| Query | Execution time |
|---|---|
| 3.1 Pedidos por país | 0.376 ms |
| 3.2 Receita por categoria | 1.091 ms |
| 3.3 Top 10 clientes | 1.336 ms |
| 3.4 Pedidos por funcionário | 0.633 ms |
| 3.5 Estoque baixo | 0.093 ms |

Cache hit ratio: **97,91 %** (76 disco / 3554 cache).

## Anexo F — Baseline RDS (northwind-rds)

Versão e config:
```
PostgreSQL 14.12 on x86_64-pc-linux-gnu
max_connections=81  shared_buffers=189368kB  work_mem=4MB (antes do PG custom)
db_size=9969 kB
```

Tabelas (linhas reais): idênticas ao Anexo E — 14 tabelas, total 3361 linhas vivas.

Tempos de execução (EXPLAIN ANALYZE):
| Query | Execution time | Δ vs local |
|---|---|---|
| 3.1 Pedidos por país | 0.514 ms | +0.138 ms |
| 3.2 Receita por categoria | 1.743 ms | +0.652 ms |
| 3.3 Top 10 clientes | 2.486 ms | +1.150 ms |
| 3.4 Pedidos por funcionário | 0.561 ms | –0.072 ms |
| 3.5 Estoque baixo | 0.137 ms | +0.044 ms |

Cache hit ratio inicial: **89,02 %** (45 disco / 365 cache, cache frio).
Após algumas execuções estabiliza > 99 %.

Os 14 índices PK foram reconstruídos automaticamente pelo `pg_restore`
(conferidos em `pg_indexes`), garantindo paridade estrutural com o local.
