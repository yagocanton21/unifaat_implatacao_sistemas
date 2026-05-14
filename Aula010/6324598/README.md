# TF10 - Migração para Amazon RDS

**Aluno:** Yago Canton
**RA:** 6324598
**Disciplina:** Implementação de Sistemas - UniFAAT

## Visão Geral

Este projeto demonstra a migração do banco de dados PostgreSQL `northwind` de um ambiente local (Docker Compose) para Amazon RDS Multi-AZ, contemplando alta disponibilidade, backup automatizado, monitoramento via CloudWatch e análise comparativa de performance e custos.

O banco original (`Aula010/docker-compose.yml`) executa PostgreSQL 14.19 em container, expondo a porta 2001 e carregando o schema Northwind (14 tabelas, ~3900 linhas SQL).

## Arquitetura

### Antes (Local)
```
+----------------------+
| Docker Host          |
|  +----------------+  |
|  | postgres-erp   |  |
|  | postgres:14.19 |  |
|  | porta 2001     |  |
|  +----------------+  |
+----------------------+
```

### Depois (RDS Multi-AZ)
```
              +------------------+
              |   Aplicação      |
              +--------+---------+
                       |
                +------v-------+
                | RDS Endpoint |
                +------+-------+
                       |
        +--------------+--------------+
        |                             |
+-------v--------+           +--------v-------+
|  RDS Primary   |  sync     |  RDS Standby   |
|  AZ us-east-1a |---------->|  AZ us-east-1b |
|  Postgres 14   |           |  Postgres 14   |
+----------------+           +----------------+
        |
        | snapshots automáticos
        v
+----------------+
|  S3 (gerido)   |
+----------------+
```

## Estrutura dos Entregáveis

```
6324598/
├── README.md                         (este arquivo)
├── migration/
│   ├── create-rds.sh                 cria RDS Multi-AZ + parameter group + SG
│   ├── migrate-data.sh               dump local + restore RDS
│   ├── validate-migration.sh         valida integridade dos dados
│   └── cleanup.sh                    remove todos recursos AWS
├── monitoring/
│   ├── cloudwatch-dashboard.json     dashboard CloudWatch
│   ├── alerts-config.json            alarmes CPU/conexões/storage/latência/billing
│   └── performance-queries.sql       queries de baseline e análise
└── docs/
    ├── migration-plan.md             plano, riscos, rollback, decisões
    ├── performance-analysis.md       benchmarks antes/depois + anexos com logs reais
    ├── cost-analysis.md              comparação de custos e TCO
    └── troubleshooting.md            problemas e soluções
```

## Como Executar a Migração

### Pré-requisitos
- AWS CLI v2 configurado (`aws configure`)
- Conta com permissões para RDS, EC2 (Security Groups), CloudWatch
- Docker em execução com `postgres-erp` ativo (`docker compose up -d`)
- `psql` e `pg_dump` instalados localmente

### Passo a Passo

1. **Exportar variáveis de ambiente**
   ```bash
   export AWS_REGION=us-east-1
   export DB_INSTANCE_ID=northwind-rds
   export DB_MASTER_USER=postgres
   export DB_MASTER_PASSWORD='<defina-uma-senha-forte>'  # nunca commitar a senha real
   export LOCAL_DB_URL='postgresql://postgres:postgres@localhost:2001/northwind'
   ```

2. **Criar instância RDS**
   ```bash
   bash migration/create-rds.sh
   ```
   Tempo aproximado: 10-15 minutos até `available`.

3. **Migrar dados**
   ```bash
   export RDS_ENDPOINT=$(aws rds describe-db-instances \
     --db-instance-identifier $DB_INSTANCE_ID \
     --query 'DBInstances[0].Endpoint.Address' --output text)
   bash migration/migrate-data.sh
   ```

4. **Validar migração**
   ```bash
   bash migration/validate-migration.sh
   ```
   Compara contagem de linhas por tabela entre local e RDS.

5. **Provisionar dashboard e alarmes**
   ```bash
   aws cloudwatch put-dashboard \
     --dashboard-name TF10-Northwind \
     --dashboard-body file://monitoring/cloudwatch-dashboard.json
   ```
   Alarmes em `monitoring/alerts-config.json` (criar via `aws cloudwatch put-metric-alarm`).

6. **Limpar recursos após avaliação**
   ```bash
   bash migration/cleanup.sh
   ```

## Resultados Obtidos

Veja `docs/performance-analysis.md` para benchmarks detalhados. Em resumo:

- Disponibilidade aumentou de SLA caseiro (~99%) para 99,95% (Multi-AZ).
- Latência média de queries OLTP comparável ao local (acréscimo ~3-8 ms de rede).
- Backup automático com retenção de 7 dias substitui scripts manuais.
- Failover testado: tempo de recuperação medido em ~60-90s.

## Custos e ROI

Análise completa em `docs/cost-analysis.md`. Resumo (db.t3.micro, 20 GB gp3, Multi-AZ, us-east-1):

| Item | Local | RDS |
|---|---|---|
| Compute (mensal) | ~R$ 80 (energia + amortização) | ~US$ 24 |
| Storage 20GB | ~R$ 10 | ~US$ 2,30 |
| Backup | manual, sem custo direto | incluso |
| Manutenção (h/mês) | 4h | ~0,5h |
| **TCO mensal estimado** | **~R$ 250** | **~US$ 30 (~R$ 165)** |

ROI positivo quando consideramos disponibilidade, backup automatizado e redução de tempo de operação.

## Evidências

Logs reais coletados durante a execução foram inlinados em `docs/performance-analysis.md`:
- **Anexo A** — validação de 14 tabelas local vs RDS (FAIL=0)
- **Anexo B** — snapshot manual `tf10-northwind-pre-tests`
- **Anexo C** — failover Multi-AZ com RTO medido (95s)
- **Anexo D** — restore do snapshot em instância temporária
- **Anexos E e F** — baselines completos local e RDS

Screenshots manuais a anexar antes do PR final:
- Console RDS com instância `available` em Multi-AZ + Parameter Group `tf10-pg14-custom`
- Dashboard CloudWatch `TF10-Northwind` com métricas após carga
- Performance Insights mostrando top queries
- Alarmes `TF10-*` em estado `OK`
