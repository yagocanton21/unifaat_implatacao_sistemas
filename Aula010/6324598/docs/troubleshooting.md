# Troubleshooting — Problemas Encontrados na Migração TF10

Esta página consolida problemas observados durante a execução do projeto e as soluções aplicadas. Serve como referência para reexecução e para apoiar a correção em caso de reprovação do PR.

## 1. `pg_dump` falha com `error: aborting because of server version mismatch`

**Causa:** versão do `pg_dump` cliente é menor do que a do servidor.

**Solução:** instalar PostgreSQL client 14.x (ou superior). No Windows, usar o instalador EDB ou o pacote do PostgreSQL via Chocolatey:
```powershell
choco install postgresql14
```
Confirmar com `pg_dump --version`.

## 2. `psql: could not connect to server: Connection refused` ao executar contra o RDS

**Causa:** Security Group bloqueia a porta 5432 vinda do IP local, ou a instância não é `publicly-accessible`.

**Diagnóstico:**
```bash
aws rds describe-db-instances --db-instance-identifier northwind-rds \
  --query 'DBInstances[0].{PubliclyAccessible:PubliclyAccessible,Endpoint:Endpoint.Address,SG:VpcSecurityGroups[].VpcSecurityGroupId}'
```

**Solução:**
1. Confirmar IP atual: `curl https://checkip.amazonaws.com`
2. Adicionar regra ingress 5432 para o IP/32:
   ```bash
   aws ec2 authorize-security-group-ingress \
     --group-id <sg-id> --protocol tcp --port 5432 --cidr <ip>/32
   ```
3. Se a instância estiver em VPC privada sem peering, criar bastion ou usar Cloud9.

## 3. `pg_restore` aborta com `ERROR: relation "<x>" already exists`

**Causa:** restore executado mais de uma vez sem limpar o banco destino.

**Solução:** `migrate-data.sh` já inclui `DROP TABLE IF EXISTS ... CASCADE` antes do restore. Se rodar manualmente, usar:
```bash
pg_restore --clean --if-exists --no-owner --no-privileges \
  --dbname=$RDS_URL backup.dump
```

## 4. Contagem de linhas diverge em `validate-migration.sh`

**Causas comuns:**
- Restore parcial (erro silencioso em `pg_restore` sem `--exit-on-error`)
- Banco de origem foi alterado entre dump e validação

**Solução:**
1. Reexecutar `migrate-data.sh` com banco origem parado (sem writes).
2. Confirmar que `--exit-on-error` está presente.
3. Comparar `pg_dump --schema-only` antes e depois para descartar diferença de DDL.

## 5. CloudWatch dashboard vazio nas primeiras horas

**Causa:** métricas RDS aparecem com 1–5 minutos de latência; algumas (Performance Insights) precisam de dados acumulados.

**Solução:** aguardar 10 minutos após a criação. Gerar carga rodando `performance-queries.sql` em loop para popular gráficos.

## 6. `aws rds delete-db-instance` retorna `InvalidParameterCombination: deletion protection enabled`

**Causa:** instância criada com `--deletion-protection`.

**Solução (já presente em `cleanup.sh`):**
```bash
aws rds modify-db-instance --db-instance-identifier northwind-rds \
  --no-deletion-protection --apply-immediately
```
Aguardar `aws rds wait db-instance-available` antes do delete.

## 7. Falha ao criar Security Group: `VPCResourceNotSpecified`

**Causa:** conta sem VPC default (raro, mas acontece em contas antigas).

**Solução:**
```bash
aws ec2 create-default-vpc
```
Ou passar `--vpc-id` explicitamente para o `create-rds.sh`.

## 8. Custos inesperados após o trabalho

**Causa principal:** instância RDS Multi-AZ esquecida ligada.

**Solução:**
1. Sempre rodar `cleanup.sh` após avaliação.
2. Conferir Cost Explorer filtrando por tag `Project=TF10`.
3. Verificar snapshots manuais órfãos:
   ```bash
   aws rds describe-db-snapshots --snapshot-type manual \
     --query 'DBSnapshots[?contains(DBSnapshotIdentifier, `northwind`)].DBSnapshotIdentifier'
   ```

## 9. Failover Multi-AZ não dispara nos testes

**Causa:** `reboot-db-instance` sem `--force-failover` reinicia na mesma AZ.

**Solução:**
```bash
aws rds reboot-db-instance \
  --db-instance-identifier northwind-rds --force-failover
```
Monitorar evento `Multi-AZ instance failover started/completed` em `aws rds describe-events`.

## 10. Parameter Group permanece em `pending-reboot`

**Causa:** `work_mem` (e outros parâmetros estáticos) só aplica após reinício.

**Solução:** disparar reboot (preferencialmente o `--force-failover` que também valida HA):
```bash
aws rds reboot-db-instance --db-instance-identifier northwind-rds --force-failover
aws rds wait db-instance-available --db-instance-identifier northwind-rds
```
Validar:
```bash
psql ... -c "SHOW work_mem;"
```

## 11. Restore de snapshot demora mais que o esperado

**Causa:** RDS provisiona nova instância do zero a partir do snapshot.
Tempo proporcional ao tamanho do storage alocado, não ao tamanho do banco.

**Solução:** snapshots de 20 GB tipicamente demoram 8-12 min. Usar
`aws rds wait db-instance-available` antes de tentar conectar. Não cancelar.

## 12. Coleção de logs

O `migrate-data.sh` gera logs nomeados em `migration/dumps/migrate_<timestamp>.log`
(diretório criado em tempo de execução; não versionado para evitar publicar
dumps binários no repositório). Os logs ficam disponíveis localmente após a
execução; o conteúdo relevante foi inlinado nos anexos de `performance-analysis.md`.
