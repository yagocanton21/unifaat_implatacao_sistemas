# Plano de Migração — Northwind Local → Amazon RDS

## Análise do Estado Atual

### Schema do Banco
- **Engine local:** PostgreSQL 14.19 (Alpine, container `postgres-erp`)
- **Banco:** `northwind`
- **Número de tabelas:** 14
  - categories, customer_customer_demo, customer_demographics, customers, employees, employee_territories, orders, order_details, products, region, shippers, suppliers, territories, us_states
- **Tamanho total estimado:** ~5 MB (banco didático)
- **Queries mais frequentes (top 5):**
  1. Pedidos por país (`customers JOIN orders` agrupado por `country`)
  2. Receita por categoria (`order_details JOIN products JOIN categories`)
  3. Top clientes por valor de compra
  4. Pedidos por funcionário
  5. Produtos abaixo do nível de reposição

### Performance Baseline (local, db.t3.micro equivalente)
- Tempo médio de resposta das 5 queries do item acima: 8–45 ms
- Conexões simultâneas em uso: 1–3 (workload acadêmico)
- IOPS: irrelevante; cache servindo > 99% (banco cabe em memória)

> Os números exatos do ambiente real estão em `performance-analysis.md`.

## Planejamento da Migração

### Engine Escolhida: **PostgreSQL 14.12** (compatível com 14.19 local)
**Justificativa:**
- Schema é nativo Postgres (`character varying`, `bytea`, `pg_dump` format custom);
- Manter mesma família elimina necessidade de conversão de tipos e DDL;
- AWS RDS oferece versão 14.x dentro do Free Tier.

### Configuração RDS
- **Classe de instância:** `db.t3.micro` (Free Tier 12 meses, 2 vCPU burst, 1 GB RAM)
- **Storage:** 20 GB gp3 (mínimo Free Tier, IOPS baseline 3000)
- **Multi-AZ:** **Sim** (exigência do TF10 — alta disponibilidade)
  > Multi-AZ dobra o custo da instância, mas é requisito da atividade.
- **Backup retention:** 7 dias (snapshots automáticos diários)
- **Performance Insights:** habilitado, retenção 7 dias (gratuito)
- **Storage Encryption:** habilitado (KMS gerido pela AWS)
- **Deletion Protection:** habilitado (proteção contra delete acidental)
- **Parameter Group customizado:** `tf10-pg14-custom` (família `postgres14`)
  - `work_mem = 8192` (8 MB) — acelera sort/hash em queries analíticas (3.2 e 3.3)
  - `log_min_duration_statement = 500` — registra slow queries acima de 500 ms
  - `log_connections = 1` / `log_disconnections = 1` — auditoria de acesso
- **Tags:** `Project=TF10`, `RA=6324598`

### Subnet Group e Posicionamento de Rede
- **DB Subnet Group:** `default-tf10` (gerado pelo `create-db-instance`), cobrindo todas as AZs da VPC default (`us-east-1a..f`).
- **Acesso:** `--publicly-accessible` habilitado, restrito por Security Group ingress TCP 5432 apenas para o IP /32 do cliente autorizado.
- **Justificativa de não usar VPC privada dedicada:**
  - Cenário didático sem aplicação dentro da AWS — todo o consumo é a partir do notebook do aluno (rede externa).
  - Subnet privada exigiria bastion EC2 ou Cloud9 (custo + complexidade fora do escopo do TF10).
  - Segurança preservada por menor privilégio na regra de SG (CIDR /32, não 0.0.0.0/0) + Encryption at Rest + senha forte do master.
  - **Roadmap produtivo:** mover para VPC privada com peering/Transit Gateway e VPC Endpoint, removendo `--publicly-accessible`. Documentado como melhoria.

### Estratégia de Migração: **Downtime curto (offline)**
**Justificativa:**
- O banco Northwind não tem tráfego de produção;
- Volume pequeno (~5 MB) → dump+restore termina em segundos;
- Solução simples, auditável, sem dependência de DMS.

#### Etapas
1. `pg_dump --format=custom` do banco local
2. `aws rds create-db-instance` (Multi-AZ, encryption, PI)
3. `pg_restore` no endpoint RDS
4. `ANALYZE` para estatísticas
5. Validação por contagem de linhas e índices

Quando faria sentido **zero-downtime** (alternativa via AWS DMS, descartada):
- Banco em produção com tráfego contínuo
- Janela de manutenção curta indisponível
- Necessidade de replicação CDC contínua durante cutover

### Plano de Rollback
1. **Aplicação ainda em produção:** já aponta para o banco local; basta reverter a string de conexão.
2. **Caso o RDS falhe após cutover:**
   - O dump original gerado por `migrate-data.sh` permanece no disco local (`migration/dumps/`), permitindo re-restore em qualquer Postgres;
   - Restaurar em qualquer Postgres local com `pg_restore`;
   - Snapshot manual antes do cutover serve como ponto de retorno na própria AWS.
3. **Caso a aplicação não conecte ao RDS:**
   - Verificar Security Group (porta 5432 liberada para o IP origem)
   - Verificar `publicly-accessible` ou roteamento VPC
   - Reverter string de conexão para localhost:2001

### Análise de Riscos

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Falha no dump local | Baixa | Alto | Validar tamanho do dump antes do restore |
| Diferença de versão Postgres | Baixa | Médio | Engine 14.x em ambos os lados |
| Security Group bloqueando 5432 | Média | Alto | Script libera IP corrente automaticamente |
| Custo estourando Free Tier | Média | Médio | Alarme `TF10-Billing-Alto` configurado |
| Multi-AZ duplicando custo | Certo | Médio | Limpar recursos com `cleanup.sh` após avaliação |
| Esquecer cleanup | Média | Alto | Billing alarm + `deletion-protection` em produção, mas removido pelo cleanup |

### Cronograma Detalhado

| Dia | Atividade | Duração |
|---|---|---|
| 1 | Documentar baseline local, criar scripts | 2h |
| 1 | Criar instância RDS via `create-rds.sh` | 15 min ativos + 15 min espera |
| 1 | Executar `migrate-data.sh` e `validate-migration.sh` | 10 min |
| 1 | Provisionar dashboard e alarmes CloudWatch | 20 min |
| 2 | Benchmarks comparativos, screenshots | 1h |
| 2 | Documentação final, PR | 1h |
| 3 | `cleanup.sh` após PR aprovado | 5 min |

### Checklist de Validação Pós-Migração

- [x] `validate-migration.sh` retorna SUCESSO (14/14 OK — `performance-analysis.md` Anexo A)
- [x] Todas as 14 tabelas presentes no RDS (contagem de linhas idêntica ao local)
- [x] Constraints e PK reconstruídos (14 índices PK confirmados em `performance-analysis.md` Anexo F)
- [x] Queries 3.1–3.5 executadas com sucesso (`performance-analysis.md` Anexos E e F)
- [x] Snapshot manual `tf10-northwind-pre-tests` criado (Anexo B)
- [x] Restore testado em instância temporária (Anexo D)
- [x] Failover Multi-AZ testado: RTO=95s (Anexo C)
- [x] Parameter Group customizado aplicado e em estado `in-sync` (work_mem=8MB, log_min_duration_statement=500ms)
- [ ] Dashboard CloudWatch exibe métricas (capturar screenshot do console)
- [ ] Alarmes em estado `OK` (capturar screenshot após carga gerada)

### RTO/RPO Alvo
- **RPO:** ≤ 5 min (backup automático + transaction logs em S3)
- **RTO:** ≤ 2 min (failover Multi-AZ medido em testes)
