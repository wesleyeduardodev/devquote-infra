# DevQuote Infrastructure

> 🚀 **GitOps:** Deploy automatizado via Argo CD - Commit e pronto!

Infraestrutura Kubernetes (K3s) do DevQuote com deploy 100% automatizado.

---

## 📋 Stack

- **K3s** - Kubernetes leve
- **Argo CD** - GitOps (deploy automatizado)
- **Traefik** - Ingress Controller
- **PostgreSQL 17** - Banco de dados
- **Redis 7** - Cache
- **Prometheus + Grafana** - Monitoramento
- **Sealed Secrets** - Secrets criptografados
- **cert-manager** - SSL automático (Let's Encrypt)

---

## 🚀 Como Funciona o Deploy (GitOps)

### **Fluxo Automatizado Completo**

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  1. Você faz commit no backend/frontend                │
│     ↓                                                   │
│  2. GitHub Actions detecta automaticamente              │
│     ↓                                                   │
│  3. Build + Testes + Docker                             │
│     ↓                                                   │
│  4. Push para Docker Hub (tag: sha-XXXXXXX)             │
│     ↓                                                   │
│  5. GitHub Actions atualiza ESTE repositório            │
│     (k8s/backend/deployment.yaml ou frontend)           │
│     ↓                                                   │
│  6. Argo CD detecta mudança no Git (~3 min)             │
│     ↓                                                   │
│  7. Aplica no cluster (rolling update)                  │
│     ↓                                                   │
│  8. Deploy completo! ✅                                  │
│                                                         │
│  ⏱️ Tempo total: ~3-5 minutos                           │
│  🔧 Comandos manuais: ZERO                              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### **Exemplo Prático**

```bash
# No projeto devquote-backend ou devquote-frontend:
git add .
git commit -m "Adiciona validação de email"
git push

# Aguardar ~3 minutos
# ✅ GitHub Actions faz build e push
# ✅ Argo CD aplica automaticamente no cluster
# ✅ Aplicação atualizada em produção!
```

**Você NÃO precisa:**
- ❌ Fazer SSH na VPS
- ❌ Rodar kubectl
- ❌ Reiniciar pods manualmente
- ❌ Fazer deploy manual

**Apenas commit → tudo acontece automaticamente!**

---

## 🔄 Rollback (Voltar Versão)

### **Quando Usar**
- Bug crítico em produção
- Versão nova apresentou problema
- Precisa voltar para versão estável

### **Como Fazer (Via Argo CD UI)**

1. Acesse: **https://devquote.com.br/argocd**
2. Login: `admin` (senha no Bitwarden)
3. Clique no app **devquote**
4. Aba **"HISTORY AND ROLLBACK"**
5. Selecione a versão anterior desejada
6. Clique **"ROLLBACK"**
7. Confirme
8. **Pronto!** Versão revertida em ~30 segundos ✅

### **O que Acontece no Rollback**
- ✅ Backend volta para tag SHA anterior
- ✅ Frontend volta para tag SHA anterior
- ✅ Rolling update (zero downtime)
- ✅ Auto-sync é desabilitado automaticamente
- ✅ Você tem tempo para corrigir o bug

### **Depois de Corrigir o Bug**

```bash
# Corrigir código localmente
git add .
git commit -m "Fix: corrige problema X"
git push

# GitHub Actions faz novo deploy automaticamente
# Argo CD aplica a versão corrigida
```

### **Status do Argo CD**

| Ícone | Status | O que significa |
|-------|--------|-----------------|
| 🟢 Synced | Sincronizado | Git = Cluster (OK) |
| 🟡 OutOfSync | Fora de sincronia | Git ≠ Cluster (normal após rollback) |
| 🔵 Syncing | Sincronizando | Aplicando mudanças... |
| 🔴 Failed | Falhou | Erro ao aplicar (ver logs) |

---

## 📧 Notificações por Email

Você recebe email automático quando:
- ❌ Deploy do backend falha
- ❌ Deploy do frontend falha

**Email configurado:** `wesleyeduardo.dev@gmail.com`

### **Exemplo de Email**

```
Assunto: [DevQuote] ❌ Deploy BACKEND falhou

⚠️ FALHA NO DEPLOY DO BACKEND

📦 Repositório: devquote-backend
🔀 Branch: main
👤 Autor: Wesley Eduardo
📝 Commit: abc1234

🔗 Ver logs: [link do GitHub Actions]
```

---

## 💾 Backup Automático PostgreSQL

### **Configuração**
- ⏰ Roda todo dia às **03:00 AM** (UTC)
- 📦 Upload para **AWS S3**: `devquote-storage/backups/postgresql/`
- 🗓️ Retenção: **últimos 7 dias**
- ✅ Totalmente automatizado (CronJob Kubernetes)

### **Ver Logs do Backup**

```bash
# Listar jobs de backup
kubectl get jobs -n devquote | grep postgres-backup

# Ver logs do último backup
kubectl logs -n devquote job/postgres-backup-XXXXXXXX
```

### **Restaurar Backup**

```bash
# 1. Baixar do S3
aws s3 cp s3://devquote-storage/backups/postgresql/devquote-backup-postgres-10-11-2025-03-00-05.sql.gz .

# 2. Descompactar
gunzip devquote-backup-postgres-10-11-2025-03-00-05.sql.gz

# 3. Restaurar no PostgreSQL
kubectl exec -it postgres-0 -n devquote -- psql -U devquote_user -d devquote < devquote-backup-postgres-10-11-2025-03-00-05.sql
```

📚 **Documentação completa:** [k8s/backup/README.md](k8s/backup/README.md)

---

## 📁 Estrutura do Repositório

```
devquote-infra/
├── README.md                    # Este arquivo
├── argocd/
│   └── application.yaml         # Argo CD Application (versionada)
└── k8s/                         # Manifestos Kubernetes
    ├── namespace.yaml
    ├── sealed-secrets.yaml      # Secrets criptografados
    ├── secrets.yaml.template    # Template de secrets
    ├── backend/
    │   ├── deployment.yaml      # ← Atualizado pelo GitHub Actions
    │   └── service.yaml
    ├── frontend/
    │   ├── deployment.yaml      # ← Atualizado pelo GitHub Actions
    │   └── service.yaml
    ├── database/                # PostgreSQL
    ├── redis/                   # Redis Cache
    ├── ingress/                 # Traefik Ingress
    ├── cert-manager/            # Let's Encrypt SSL
    ├── monitoring/              # Prometheus + Grafana
    ├── backup/                  # CronJob de backup
    │   ├── README.md
    │   ├── cronjob.yaml
    │   └── configmap.yaml
    └── argocd/                  # Argo CD Notifications (opcional)
        ├── README.md
        └── argocd-notifications-cm.yaml
```

---

## 🔧 Comandos Úteis

### **Ver Status dos Pods**

```bash
# Listar todos os pods
kubectl get pods -n devquote

# Acompanhar em tempo real
kubectl get pods -n devquote -w
```

### **Ver Logs**

```bash
# Backend
kubectl logs -f deployment/backend -n devquote

# Frontend
kubectl logs -f deployment/frontend -n devquote

# PostgreSQL
kubectl logs -f postgres-0 -n devquote
```

### **Ver Eventos Recentes**

```bash
kubectl get events -n devquote --sort-by='.lastTimestamp'
```

### **Verificar Recursos (CPU/RAM)**

```bash
kubectl top pods -n devquote
```

---

## 🔗 Links Importantes

- **Aplicação:** https://devquote.com.br
- **Argo CD:** https://devquote.com.br/argocd
- **Grafana:** https://devquote.com.br/grafana
- **Swagger API:** https://devquote.com.br/swagger-ui

---

## ⚠️ Regras Importantes

### **✅ O QUE FAZER**
- ✅ Fazer mudanças via **Git** (commit → push)
- ✅ Usar **Argo CD UI** para rollback
- ✅ Ver logs via `kubectl logs`
- ✅ Monitorar via Grafana

### **❌ O QUE NÃO FAZER**
- ❌ **NUNCA** fazer `kubectl apply` manual nos deployments
- ❌ **NUNCA** editar recursos diretamente no cluster (`kubectl edit`)
- ❌ **NUNCA** fazer `kubectl rollout restart` manual
- ❌ **NUNCA** fazer `kubectl rollout undo`

**Por quê?**
- Git é a fonte da verdade (GitOps)
- Mudanças manuais serão sobrescritas pelo Argo CD
- Perde-se rastreabilidade e histórico

---

## 🎯 Resumo do Fluxo

| Tarefa | Como fazer |
|--------|------------|
| **Deploy** | Commit → Push (automático) |
| **Rollback** | Argo CD UI → History → Rollback |
| **Ver logs** | `kubectl logs -f deployment/backend -n devquote` |
| **Monitorar** | Grafana (https://devquote.com.br/grafana) |
| **Backup** | Automático (3h AM, S3) |
| **Restaurar** | `kubectl exec ... psql < backup.sql` |

---

## 📊 Recursos Alocados

| Componente | Réplicas | RAM | CPU |
|------------|----------|-----|-----|
| Backend | 1 | 512-750Mi | 400-800m |
| Frontend | 1 | 128-256Mi | 100-200m |
| PostgreSQL | 1 | 256-512Mi | 250-500m |
| Redis | 1 | 128-256Mi | 100-200m |

---

## 🔄 Migração de Configurações (2025-01-16)

### **Mudança Arquitetural Importante**

A maioria das variáveis de configuração **migrou do Kubernetes para o banco de dados** (tabela `system_parameter`).

**Apenas 10 variáveis permanecem no Kubernetes:**

```yaml
# PostgreSQL (6) - necessárias para o backend conectar ao banco
POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD
SPRING_DATASOURCE_URL, SPRING_DATASOURCE_USERNAME, SPRING_DATASOURCE_PASSWORD

# AWS S3 (4) - necessárias para o CronJob de backup funcionar
AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_S3_BUCKET_NAME, AWS_S3_REGION
```

**Todas as demais configurações** (JWT, Email, CORS, multipart, etc) **agora estão no banco**.

**Vantagens:**
- ✅ Configurações alteráveis via interface web (sem redeploy)
- ✅ Auditoria de mudanças
- ✅ Valores sensíveis criptografados no banco
- ✅ Menos secrets no Kubernetes

**Detalhes:** Ver [SECRETS.md](./SECRETS.md#-mudança-arquitetural-2025-01-16)

---

## 📚 Documentação Complementar

- **Secrets:** [SECRETS.md](./SECRETS.md) - Como gerenciar secrets e Sealed Secrets
- **Arquitetura:** [architecture.md](./architecture.md) - Visão técnica completa
- **Backup:** [k8s/backup/README.md](k8s/backup/README.md) - Detalhes do backup automático

---

**Última atualização:** 2025-01-16
**Versão:** GitOps 2.0 (Deploy + Rollback automatizado) + Config em banco
