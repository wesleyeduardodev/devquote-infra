# DevQuote Infrastructure

> 🔐 **Secrets:** Ver [SECRETS.md](./SECRETS.md) para gerenciamento de credenciais

Infraestrutura Kubernetes (K3s) para DevQuote com **GitOps via Argo CD**.

---

## Stack

- **K3s** - Kubernetes leve
- **Traefik** - Ingress Controller + Load Balancer
- **PostgreSQL 17** - Banco de dados
- **Redis 7** - Cache distribuído
- **Prometheus + Grafana** - Observabilidade
- **Argo CD** - GitOps (Continuous Delivery)
- **Sealed Secrets** - Gerenciamento seguro de secrets
- **cert-manager** - Certificados SSL/TLS (Let's Encrypt)

---

## Estrutura

```
devquote-infra/
├── README.md
├── SECRETS.md                 # Guia de secrets
├── argocd/
│   └── application.yaml       # Argo CD Application (versionada)
├── k8s/
│   ├── namespace.yaml
│   ├── sealed-secrets.yaml    # ✅ Encrypted (Git)
│   ├── argocd/                # Ingress Argo CD
│   ├── backend/               # Deployment + Service
│   ├── frontend/              # Deployment + Service
│   ├── database/              # PostgreSQL (StatefulSet + PVC)
│   ├── redis/                 # Redis (StatefulSet)
│   ├── cert-manager/          # Let's Encrypt ClusterIssuer
│   ├── ingress/               # Ingress principal (Traefik)
│   └── monitoring/            # Prometheus + Grafana
│       ├── grafana/
│       └── prometheus/
└── architecture.md            # Documentação técnica
```

---

## 🚀 GitOps: Como Funciona o Deploy

### **Fluxo Completo (Automatizado)**

```
1. Dev faz commit no devquote-backend ou devquote-frontend
   ↓
2. GitHub Actions detecta mudança automaticamente
   ↓
3. Build + Lint/Tests
   ↓
4. Docker build com tag SHA imutável (sha-XXXXXXX)
   ↓
5. Push para Docker Hub (3 tags: sha-XXX, latest, version)
   ↓
6. GitHub Actions clona devquote-infra
   ↓
7. Atualiza k8s/backend/deployment.yaml ou k8s/frontend/deployment.yaml
   ↓
8. Commit + Push para devquote-infra
   ↓
9. Argo CD detecta mudança no Git (auto-sync)
   ↓
10. Aplica rolling update no Kubernetes (zero downtime)
   ↓
11. Deploy completo ✅
```

**Você não precisa fazer NADA manualmente!** Apenas commitar código.

### **Exemplo Prático**

```bash
# No projeto devquote-backend:
git add .
git commit -m "Fix bug no cálculo de faturamento"
git push

# Aguardar ~3 minutos:
# ✅ GitHub Actions: Build + Docker push
# ✅ Infra repo atualizado automaticamente
# ✅ Argo CD aplica no cluster
# ✅ Pod antigo → Pod novo (rolling update)
```

---

## 🔄 Rollback Via Argo CD UI

### **Como Fazer Rollback**

1. Acesse: **https://devquote.com.br/argocd**
2. Faça login (user: `admin`)
3. Clique no app **devquote**
4. Aba **"HISTORY AND ROLLBACK"**
5. Veja todas as versões anteriores (até 10 versões)
6. Selecione a versão desejada
7. Clique **"ROLLBACK"**
8. Confirme
9. Argo CD reverte automaticamente ✅

**O rollback é seguro:**
- ✅ Reverte para tag SHA específica (imutável)
- ✅ Rolling update sem downtime
- ✅ Git é atualizado automaticamente
- ✅ Histórico completo auditável

---

## 📦 Deploy Inicial (Primeira Vez)

### 1. Criar Argo CD Application

```bash
# Aplicar a Application do Argo CD (versionada em Git)
kubectl apply -f argocd/application.yaml
```

Isso cria a Application `devquote` que gerencia todos os recursos em `k8s/`.

### 2. Verificar Deploy

```bash
# Ver Application no Argo CD
kubectl get application devquote -n argocd

# Ver todos os recursos gerenciados
kubectl get all -n devquote

# Acessar UI
https://devquote.com.br/argocd
```

**Pronto!** Argo CD sincroniza automaticamente tudo do Git.

---

## 🔧 Comandos Úteis

### Logs

```bash
# Backend
kubectl logs -f deployment/backend -n devquote

# Frontend
kubectl logs -f deployment/frontend -n devquote

# PostgreSQL
kubectl logs -f postgres-0 -n devquote

# Redis
kubectl logs -f redis-0 -n devquote
```

### Monitoramento

```bash
# Ver todos os pods
kubectl get pods -n devquote

# Com atualização em tempo real (watch)
kubectl get pods -n devquote -w

# Ver serviços
kubectl get svc -n devquote

# Ver eventos recentes
kubectl get events -n devquote --sort-by='.lastTimestamp'

# Ver uso de recursos
kubectl top pods -n devquote
```

### Argo CD

```bash
# Ver status da Application
kubectl get application devquote -n argocd

# Forçar sync manual (se necessário)
kubectl -n argocd patch application devquote -p '{"operation":{"sync":{"revision":"HEAD"}}}' --type merge

# Ver histórico de deploys
kubectl get application devquote -n argocd -o jsonpath='{.status.history}'
```

### Database

```bash
# Conectar no PostgreSQL
kubectl exec -it postgres-0 -n devquote -- psql -U devquote_user -d devquote

# Backup manual
kubectl exec -n devquote postgres-0 -- pg_dump -U devquote_user devquote > backup.sql

# Verificar dados
kubectl exec -it postgres-0 -n devquote -- psql -U devquote_user -d devquote -c "SELECT COUNT(*) FROM users;"
```

---

## 📊 Redis Cache

### Verificar Chaves no Redis

```bash
# Conectar no pod do Redis
kubectl exec -it redis-0 -n devquote -- redis-cli

# Dentro do redis-cli:
KEYS *                    # Listar todas as chaves
GET "projects::1"         # Ver conteúdo de uma chave
TTL "projects::1"         # Ver tempo de expiração (segundos)
FLUSHALL                  # Limpar todas as chaves (cuidado!)
exit                      # Sair

# Comandos diretos (sem entrar no redis-cli):
kubectl exec -it redis-0 -n devquote -- redis-cli KEYS "*"
kubectl exec -it redis-0 -n devquote -- redis-cli GET "projects::1"
kubectl exec -it redis-0 -n devquote -- redis-cli TTL "projects::1"
kubectl exec -it redis-0 -n devquote -- redis-cli INFO
kubectl exec -it redis-0 -n devquote -- redis-cli MONITOR

# Ver logs do Redis
kubectl logs -f redis-0 -n devquote
```

**Cache configurado:**
- TTL: 10 minutos
- Método cacheado: `findById()` em `ProjectServiceImpl`

---

## 📈 Recursos Alocados

| Componente | Réplicas | RAM | CPU |
|------------|----------|-----|-----|
| Backend | 1 | 512-750Mi | 400-800m |
| Frontend | 1 | 128-256Mi | 100-200m |
| PostgreSQL | 1 | 256-512Mi | 250-500m |
| Redis | 1 | 128-256Mi | 100-200m |
| Prometheus | 1 | 256-512Mi | 100-200m |
| Grafana | 1 | 128-256Mi | 50-100m |

---

## 🔗 Links Úteis

- **Aplicação:** https://devquote.com.br
- **Argo CD:** https://devquote.com.br/argocd
- **Grafana:** https://devquote.com.br/grafana (user: `admin`, senha: `admin123`)
- **API Docs:** https://devquote.com.br/swagger-ui

---

## 📚 Documentação Adicional

- [architecture.md](architecture.md) - Visão geral da arquitetura
- [SECRETS.md](./SECRETS.md) - Gerenciamento de secrets e Sealed Secrets

---

## ⚠️ Observações Importantes

- **NÃO faça kubectl apply manual** nos deployments - use GitOps via Argo CD
- **NÃO altere recursos diretamente no cluster** - faça mudanças no Git
- **Backup automático** do PostgreSQL roda diariamente às 02:00 (via cron)
- **Certificado SSL** renovado automaticamente pelo cert-manager
- **Rollback sempre via Argo CD UI** - nunca use kubectl rollout undo
