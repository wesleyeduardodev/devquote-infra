# DevQuote Infrastructure

> 🔐 **Secrets:** Ver [SECRETS.md](./SECRETS.md) para gerenciamento de credenciais

Infraestrutura Kubernetes (K3s) para DevQuote com GitOps via Argo CD.

---

## Stack

- **K3s** - Kubernetes leve
- **Traefik** - Ingress Controller
- **PostgreSQL 17** - Banco de dados
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

# Database
kubectl apply -f k8s/database/

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

# Escalar
kubectl scale deployment backend --replicas=3 -n devquote

# Rollback
kubectl rollout undo deployment/backend -n devquote

# Recursos
kubectl top pods -n devquote
```

---

## Recursos

| Componente | Réplicas | RAM | CPU |
|------------|----------|-----|-----|
| Backend | 2 | 512-750Mi | 300-500m |
| Frontend | 1 | 128-256Mi | 100-200m |
| PostgreSQL | 1 | 256-512Mi | 250-500m |
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
