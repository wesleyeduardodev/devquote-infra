# 🗄️ Backup Automático PostgreSQL

Backup diário do banco de dados PostgreSQL para AWS S3 via Kubernetes CronJob.

---

## 📋 O que faz

- ✅ Backup automático **todos os dias às 3h AM** (UTC)
- ✅ Upload direto para **AWS S3** (bucket configurado nos secrets)
- ✅ Compactação gzip (economiza espaço)
- ✅ Retenção: **últimos 7 dias** (backups antigos são deletados automaticamente)
- ✅ Logs centralizados no Kubernetes

---

## 🚀 Deploy

### 1. Aplicar os manifestos

```bash
kubectl apply -f k8s/backup/
```

Isso criará:
- **ConfigMap**: Script de backup + upload S3
- **CronJob**: Agendamento diário às 3h AM

### 2. Verificar se foi criado

```bash
kubectl get cronjob -n devquote
```

**Saída esperada:**
```
NAME              SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
postgres-backup   0 3 * * *   False     0        <none>          10s
```

---

## 🔍 Monitoramento

### Ver próxima execução
```bash
kubectl get cronjob postgres-backup -n devquote
```

### Ver histórico de jobs executados
```bash
kubectl get jobs -n devquote | grep postgres-backup
```

### Ver logs do último backup
```bash
kubectl logs -n devquote job/postgres-backup-<timestamp>
```

**Exemplo:**
```bash
kubectl logs -n devquote job/postgres-backup-28998765
```

---

## 🧪 Testar Backup Manualmente (Sem Esperar 3h AM)

```bash
kubectl create job -n devquote postgres-backup-manual --from=cronjob/postgres-backup
```

Acompanhar logs:
```bash
kubectl logs -n devquote -f job/postgres-backup-manual
```

---

## 📦 Onde ficam os backups

**AWS S3:**
```
s3://<seu-bucket>/backups/postgresql/devquote_backup_YYYYMMDD_HHMMSS.sql.gz
```

**Exemplo:**
```
s3://devquote-storage/backups/postgresql/devquote_backup_20250110_030015.sql.gz
```

---

## ♻️ Retenção de Backups

O script mantém **apenas os últimos 7 dias** de backup no S3.

Backups mais antigos são **automaticamente deletados** após cada execução bem-sucedida.

---

## ⚠️ Desabilitar Script Antigo da VPS

**IMPORTANTE:** Após ativar este CronJob Kubernetes, desabilite o script antigo da VPS para evitar backups duplicados.

### Na VPS, execute:

```bash
# Editar crontab
crontab -e
```

**Comentar ou remover a linha do backup:**
```bash
# 0 3 * * * /path/to/backup-postgres.sh  # DESABILITADO - Usando CronJob K8s
```

Ou remover completamente:
```bash
crontab -r  # Remove TODOS os cron jobs (cuidado!)
```

Para verificar se foi desabilitado:
```bash
crontab -l
```

---

## 🔧 Configurações

### Alterar horário do backup

Editar `cronjob.yaml` linha 9:

```yaml
schedule: "0 3 * * *"  # 3h AM UTC
```

**Exemplos:**
- `"0 2 * * *"` = 2h AM UTC
- `"30 4 * * *"` = 4h30 AM UTC
- `"0 3 * * 0"` = 3h AM UTC apenas aos domingos

**Aplicar mudança:**
```bash
kubectl apply -f k8s/backup/cronjob.yaml
```

---

### Alterar retenção de backups

Editar `configmap.yaml` linha 15:

```bash
RETENTION_DAYS=7  # Manter últimos 7 dias
```

**Aplicar mudança:**
```bash
kubectl apply -f k8s/backup/configmap.yaml
```

---

## 🛠️ Restauração de Backup

### 1. Baixar backup do S3

```bash
aws s3 cp s3://devquote-storage/backups/postgresql/devquote_backup_20250110_030015.sql.gz .
```

### 2. Descompactar

```bash
gunzip devquote_backup_20250110_030015.sql.gz
```

### 3. Restaurar no PostgreSQL

**Via kubectl (dentro do cluster):**
```bash
kubectl exec -it -n devquote postgres-0 -- psql -U devquote_user -d devquote < devquote_backup_20250110_030015.sql
```

**Ou via pod temporário:**
```bash
kubectl run -n devquote restore-temp --image=postgres:17-alpine --rm -it -- \
  psql -h postgres-service -U devquote_user -d devquote < devquote_backup_20250110_030015.sql
```

---

## 📌 Requisitos

- ✅ AWS S3 Bucket configurado
- ✅ Credenciais AWS nos secrets (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_BUCKET_NAME`, `AWS_S3_REGION`)
- ✅ PostgreSQL rodando no cluster (service: `postgres-service`)
- ✅ Credenciais PostgreSQL nos secrets (`POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`)

Tudo isso **já está configurado** no `sealed-secrets.yaml`! 🎉

---

## ❓ Troubleshooting

### Backup não está rodando

1. Verificar se CronJob existe:
```bash
kubectl get cronjob -n devquote
```

2. Verificar logs do último job:
```bash
kubectl logs -n devquote job/postgres-backup-<timestamp>
```

3. Verificar eventos:
```bash
kubectl get events -n devquote --sort-by='.lastTimestamp' | grep postgres-backup
```

### Erro de upload para S3

Verificar credenciais AWS:
```bash
kubectl get secret devquote-secrets -n devquote -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d
```

### Job ficou travado

Deletar job manualmente:
```bash
kubectl delete job -n devquote postgres-backup-<timestamp>
```

---

**Última atualização:** 2025-11-10
