# Gerenciamento de Secrets

Este projeto usa **Sealed Secrets** para versionamento seguro de credenciais.

---

## Como Funciona

```
secrets.yaml (local, texto claro)
      ↓ kubeseal (criptografa)
sealed-secrets.yaml (criptografado)
      ↓ git push
GitHub (seguro)
      ↓ Argo CD
Kubernetes Secret
      ↓
Pods (variáveis de ambiente)
```

**⚠️ NUNCA commitar `k8s/secrets.yaml` (texto claro)**
**✅ SEMPRE commitar `k8s/sealed-secrets.yaml` (criptografado)**

---

## 🔄 Mudança Arquitetural (2025-01-16)

**IMPORTANTE:** A maioria das variáveis de configuração **migrou do Kubernetes para o banco de dados** (tabela `system_parameter`).

### Variáveis que PERMANECEM no Kubernetes (10 variáveis)

Apenas **configurações de infraestrutura** que o backend precisa **antes** de acessar o banco:

```yaml
# Database PostgreSQL (3 variáveis)
POSTGRES_DB: devquote
POSTGRES_USER: devquote_user
POSTGRES_PASSWORD: <senha>

# Spring DataSource (3 variáveis)
SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-service:5432/devquote?sslmode=disable
SPRING_DATASOURCE_USERNAME: devquote_user
SPRING_DATASOURCE_PASSWORD: <senha>

# AWS S3 para Backup Automático PostgreSQL (4 variáveis)
AWS_ACCESS_KEY_ID: AKIA...
AWS_SECRET_ACCESS_KEY: <secret-key>
AWS_S3_BUCKET_NAME: devquote-storage
AWS_S3_REGION: us-east-1
```

**Motivo AWS no Kubernetes:**
- O CronJob de backup PostgreSQL roda **fora do contexto do backend**
- Não tem acesso ao banco para buscar credenciais
- Necessita das credenciais AWS para enviar backups ao S3

### Variáveis que MIGRARAM para o Banco

Todas as configurações da **aplicação** agora estão em `system_parameter`:

```
APP_JWTSECRET (criptografada)
DEVQUOTE_CORS_ALLOWED_ORIGINS
DEVQUOTE_EMAIL_ENABLED
DEVQUOTE_EMAIL_FROM
MAIL_HOST
MAIL_PORT
MAIL_USERNAME
MAIL_PASSWORD (criptografada)
MAIL_SMTP_AUTH
MAIL_SMTP_STARTTLS_ENABLE
MAIL_SMTP_SSL_TRUST
APP_JWTEXPIRATIONMS
SPRING_SERVLET_MULTIPART_MAX_FILE_SIZE
SPRING_SERVLET_MULTIPART_MAX_REQUEST_SIZE
SERVER_ERROR_INCLUDE_MESSAGE
... e outras
```

**Vantagens da migração:**
- ✅ Configurações alteráveis via interface web (sem redeploy)
- ✅ Auditoria de mudanças
- ✅ Valores criptografados no banco (campos sensíveis)
- ✅ Menos secrets no Kubernetes

---

## Alterar uma Secret

### 1. Editar arquivo local

```bash
vim k8s/secrets.yaml
```

Exemplo:
```yaml
stringData:
  POSTGRES_PASSWORD: NovaSenha123!
```

### 2. Re-criptografar

```bash
~/bin/kubeseal.exe --format=yaml \
  --cert=.sealed-secrets/public-key.pem \
  < k8s/secrets.yaml \
  > k8s/sealed-secrets.yaml
```

### 3. Commitar

```bash
git add k8s/sealed-secrets.yaml
git commit -m "Update PostgreSQL password"
git push
```

### 4. Aguardar Argo CD (1-3 min)

Argo CD aplica automaticamente.

### 5. Reiniciar pods (se necessário)

```bash
kubectl rollout restart deployment/backend -n devquote
```

---

## Adicionar Nova Secret

### 1. Editar secrets.yaml

```yaml
stringData:
  # ... secrets existentes ...
  NEW_API_KEY: abc123xyz789
```

### 2. Re-criptografar + Commitar

Mesmos passos acima (2-5).

---

## Remover uma Secret

### 1. Editar secrets.yaml

Remover a linha da variável.

### 2. Re-criptografar + Commitar

Mesmos passos acima (2-5).

### 3. Atualizar código

Remover referências à variável no código da aplicação.

---

## Troubleshooting

### Secret não está sendo atualizado

```bash
# Verificar Argo CD
kubectl get application devquote -n argocd

# Forçar sync
kubectl patch application devquote -n argocd \
  --type merge \
  -p '{"operation":{"sync":{"revision":"main"}}}'

# Verificar SealedSecret
kubectl get sealedsecret -n devquote

# Ver logs do controller
kubectl logs -n kube-system -l app.kubernetes.io/name=sealed-secrets
```

### Pods não estão lendo a nova secret

```bash
# Reiniciar pods
kubectl rollout restart deployment/backend -n devquote
```

### Ver valor de uma secret (⚠️ CUIDADO)

```bash
# Listar chaves
kubectl get secret devquote-secrets -n devquote -o jsonpath='{.data}' | grep -o '"[^"]*":'

# Ver valor (base64)
kubectl get secret devquote-secrets -n devquote -o jsonpath='{.data.POSTGRES_PASSWORD}'

# Descriptografar
kubectl get secret devquote-secrets -n devquote -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d
```

---

## Estrutura de Arquivos

```
devquote-infra/
├── .sealed-secrets/
│   └── public-key.pem         # Local only
├── k8s/
│   ├── secrets.yaml           # ⚠️ NUNCA commitar
│   └── sealed-secrets.yaml    # ✅ Commitar (criptografado)
└── .gitignore
    └── k8s/secrets.yaml       # Protegido
```

---

## Boas Práticas

### ✅ Fazer

- Sempre editar `secrets.yaml` localmente
- Sempre re-criptografar após mudanças
- Commit apenas `sealed-secrets.yaml`
- Rotacionar senhas periodicamente
- Usar senhas fortes (mínimo 16 caracteres)

### ❌ Não Fazer

- NUNCA commitar `secrets.yaml` (texto claro)
- NUNCA compartilhar senhas em Slack/email
- NUNCA usar senhas fracas
- NUNCA expor secrets em logs

---

## 💾 Backup Manual do PostgreSQL

### Criar Backup Manual e Enviar para S3

Quando precisar fazer um backup manual (fora do agendamento automático das 3h AM):

#### 1. Criar Job Manual de Backup

```bash
# Criar job a partir do CronJob existente
ssh vps "kubectl create job --from=cronjob/postgres-backup backup-manual-$(date +%s) -n devquote"
```

#### 2. Acompanhar Execução

```bash
# Verificar status do pod
ssh vps "kubectl get pods -n devquote | grep backup-manual"

# Ver logs em tempo real
ssh vps "kubectl logs -f <nome-do-pod> -n devquote"
```

#### 3. Verificar Sucesso

Logs de sucesso devem mostrar:
```
✓ Backup created: devquote-backup-postgres-DD-MM-YYYY-HH-MM-SS.sql.gz (XX.XK)
✓ Backup uploaded successfully to S3
✓ Local backup file removed
Backup completed successfully!
```

#### 4. Limpar Job Após Conclusão

```bash
# Deletar job de teste (opcional, será removido automaticamente após 1h)
ssh vps "kubectl delete job <nome-do-job> -n devquote"
```

#### 5. Verificar Arquivo no S3

Acesse o bucket S3 ou use AWS CLI:
```bash
aws s3 ls s3://devquote-storage/backups/postgresql/ --profile devquote
```

### Informações do Backup

- **Localização S3**: `s3://devquote-storage/backups/postgresql/`
- **Formato do arquivo**: `devquote-backup-postgres-DD-MM-YYYY-HH-MM-SS.sql.gz`
- **Retenção**: 7 dias (limpeza automática)
- **Tamanho médio**: ~50KB (comprimido com gzip)
- **Storage Class**: STANDARD_IA (Infrequent Access)

### Restaurar Backup

```bash
# 1. Baixar do S3
aws s3 cp s3://devquote-storage/backups/postgresql/devquote-backup-postgres-DD-MM-YYYY-HH-MM-SS.sql.gz . --profile devquote

# 2. Descompactar
gunzip devquote-backup-postgres-DD-MM-YYYY-HH-MM-SS.sql.gz

# 3. Restaurar no PostgreSQL
ssh vps "kubectl exec -it postgres-0 -n devquote -- psql -U devquote_user -d devquote < devquote-backup-postgres-DD-MM-YYYY-HH-MM-SS.sql"
```

---

## Referências

- [Sealed Secrets GitHub](https://github.com/bitnami-labs/sealed-secrets)
- [Documentação Oficial](https://sealed-secrets.netlify.app/)
