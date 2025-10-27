# DevQuote Infrastructure

> 🔐 **Secrets:** Ver [SECRETS.md](./SECRETS.md) para gerenciamento de credenciais

Infraestrutura Kubernetes (K3s) para DevQuote com GitOps via Argo CD.

---

## Stack

- **K3s** - Kubernetes leve
- **Traefik** - Ingress Controller
- **PostgreSQL 17** - Banco de dados
- **Redis 7** - Cache
- **Prometheus + Grafana** - Observabilidade
- **Argo CD** - GitOps
- **Sealed Secrets** - Gerenciamento seguro de secrets

---

## Estrutura

```
devquote-infra/
├── README.md
├── SECRETS.md                 # Guia de secrets
├── k8s/
│   ├── namespace.yaml
│   ├── secrets.yaml           # ⚠️ Local only
│   ├── sealed-secrets.yaml    # ✅ Encrypted (Git)
│   ├── backend/
│   ├── frontend/
│   ├── database/
│   ├── redis/
│   ├── ingress/
│   └── monitoring/
└── docs/
    └── architecture.md
```

---

## Quick Start

### 1. Configurar Secrets

```bash
# Editar secrets localmente
vim k8s/secrets.yaml

# Criptografar
~/bin/kubeseal.exe --format=yaml \
  --cert=.sealed-secrets/public-key.pem \
  < k8s/secrets.yaml \
  > k8s/sealed-secrets.yaml

# Commitar (apenas o criptografado)
git add k8s/sealed-secrets.yaml
git commit -m "Update secrets"
git push
```

### 2. Deploy Inicial

```bash
# Namespace + Secrets
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/sealed-secrets.yaml

# Database + Redis
kubectl apply -f k8s/database/
kubectl apply -f k8s/redis/

# Backend + Frontend
kubectl apply -f k8s/backend/
kubectl apply -f k8s/frontend/

# Ingress
kubectl apply -f k8s/ingress/

# Monitoring (opcional)
kubectl apply -f k8s/monitoring/
```

### 3. Verificar

```bash
kubectl get pods -n devquote
kubectl get ingress -n devquote
```

---

## Atualizar Aplicação

### Via GitOps (Recomendado)

```bash
# 1. Backend foi buildado e gerou nova imagem
# 2. Argo CD detecta mudança automaticamente
# 3. Rolling update sem downtime
```

### Manual

```bash
kubectl rollout restart deployment/backend -n devquote
```

---

## Comandos Úteis

```bash
# Logs
kubectl logs -f deployment/backend -n devquote

# Ver todos os pods
kubectl get pods -n devquote

# Com atualização em tempo real (watch)
kubectl get pods -n devquote -w

# Ver serviços
kubectl get svc -n devquote

# Ver eventos recentes
kubectl get events -n devquote --sort-by='.lastTimestamp'

# Escalar
kubectl scale deployment backend --replicas=3 -n devquote

# Rollback
kubectl rollout undo deployment/backend -n devquote

# Ver uso de recursos
kubectl top pods -n devquote
```

---

## Redis Cache

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

## Recursos

| Componente | Réplicas | RAM | CPU |
|------------|----------|-----|-----|
| Backend | 1 | 512-750Mi | 400-800m |
| Frontend | 1 | 128-256Mi | 100-200m |
| PostgreSQL | 1 | 256-512Mi | 250-500m |
| Redis | 1 | 128-256Mi | 100-200m |
| Prometheus | 1 | 256-512Mi | 100-200m |
| Grafana | 1 | 128-256Mi | 50-100m |

---

## Documentação

- [Architecture](./docs/architecture.md) - Visão geral da arquitetura
- [SECRETS.md](./SECRETS.md) - Gerenciamento de secrets

---

## Links

- **Aplicação:** https://devquote.com.br
- **Grafana:** https://devquote.com.br/grafana
- **Argo CD:** https://devquote.com.br/argocd
