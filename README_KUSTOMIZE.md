# DevQuote Infrastructure - GitOps Profissional

> 🎯 **Kustomize + Argo CD + Tags SHA Imutáveis**

Infraestrutura Kubernetes (K3s) para DevQuote com **GitOps completo**, seguindo as **melhores práticas da CNCF**.

---

## 🏗️ Stack

- **K3s** - Kubernetes leve
- **Kustomize** - Gerenciamento de manifests (base + overlays)
- **Argo CD** - GitOps (auto-sync, self-heal, rollback)
- **Traefik** - Ingress Controller + Load Balancer
- **PostgreSQL 17** - Banco de dados
- **Redis 7** - Cache
- **Prometheus + Grafana** - Observabilidade
- **Sealed Secrets** - Gerenciamento seguro de credenciais

---

## 📂 Estrutura

```
devquote-infra/
├── README.md                         # Este arquivo
├── KUSTOMIZE_MIGRATION.md            # Guia completo de migração
├── ROLLBACK_GUIDE.md                 # Guia rápido de rollback
├── SECRETS.md                        # Gerenciamento de secrets
│
├── base/                             # ✨ Configuração base (Kustomize)
│   ├── kustomization.yaml
│   ├── backend/
│   │   ├── deployment.yaml           # Sem tag (sobrescrita pelo overlay)
│   │   └── service.yaml
│   └── frontend/
│       ├── deployment.yaml
│       └── service.yaml
│
├── overlays/                         # ✨ Sobrescritas por ambiente
│   └── production/
│       └── kustomization.yaml        # Tags SHA atualizadas pelo CI/CD
│
├── argocd/                           # ✨ Configuração Argo CD
│   └── application.yaml
│
└── k8s/                              # Resources compartilhados
    ├── namespace.yaml
    ├── sealed-secrets.yaml
    ├── backend/                      # [DEPRECATED] Usar base/ agora
    ├── frontend/                     # [DEPRECATED] Usar base/ agora
    ├── database/
    ├── redis/
    ├── ingress/
    └── monitoring/
```

---

## 🚀 Quick Start

### **1. Pré-requisitos**

```bash
# Instalar Kustomize (se necessário)
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
mv kustomize ~/bin/

# Verificar instalação
kustomize version
```

### **2. Configurar GitHub Personal Access Token**

```bash
# 1. Criar PAT: https://github.com/settings/tokens/new
# Permissões: repo (Full control)

# 2. Adicionar nos repositórios backend e frontend:
# Settings → Secrets → Actions → New repository secret
# Nome: GH_PAT
# Valor: ghp_xxxxxxxxxxxx
```

### **3. Deploy Inicial**

```bash
# Namespace + Secrets
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/sealed-secrets.yaml

# Database + Redis
kubectl apply -f k8s/database/
kubectl apply -f k8s/redis/

# Aplicar usando Kustomize (production overlay)
kubectl apply -k overlays/production/

# Ingress
kubectl apply -f k8s/ingress/

# Monitoring (opcional)
kubectl apply -f k8s/monitoring/
```

### **4. Configurar Argo CD**

```bash
# Aplicar Application
kubectl apply -f argocd/application.yaml

# Verificar
kubectl get application -n argocd

# Status
kubectl describe application devquote -n argocd

# UI: https://devquote.com.br/argocd
# Configurar: Auto-Sync ✓ | Self-Heal ✓ | Prune ✓
```

---

## 🔄 Fluxo GitOps

### **Deploy Automático**

```
Developer
   ↓ git push (backend ou frontend)
GitHub Actions
   ✓ Build app
   ✓ Build Docker image: sha-a1b2c3d
   ✓ Push Docker Hub
   ✓ Clone devquote-infra
   ✓ kustomize edit set image ...sha-a1b2c3d
   ✓ git commit + push
   ↓
Argo CD
   ✓ Detecta commit no Git
   ✓ Diff: old SHA → new SHA
   ✓ Apply no cluster (rolling update)
   ✓ Health check
   ↓
Kubernetes
   ✓ Backend: sha-a1b2c3d (PROD)
   ✓ Zero downtime
```

### **Como funciona:**

1. **Você:** Faz commit no `devquote-backend` ou `devquote-frontend`
2. **GitHub Actions:** Gera imagem com tag SHA e atualiza `overlays/production/kustomization.yaml`
3. **Argo CD:** Detecta mudança no Git e aplica no cluster
4. **Resultado:** Deploy automático em ~2 minutos

---

## 🔙 Rollback

### **Via Argo CD UI (Recomendado)**

```
1. Acesse: https://devquote.com.br/argocd
2. devquote → HISTORY AND ROLLBACK
3. Selecione revisão anterior
4. ROLLBACK

⏱️ Tempo: 30 segundos
```

### **Via Git Revert**

```bash
cd devquote-infra
git log --oneline overlays/production/kustomization.yaml
git revert COMMIT_HASH --no-commit
git commit -m "Rollback to previous version"
git push
```

> 📖 **Ver guia completo:** [ROLLBACK_GUIDE.md](./ROLLBACK_GUIDE.md)

---

## 🛠️ Comandos Úteis

### **Ver versão em produção**

```bash
# Backend
kubectl get deployment backend -n devquote \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Frontend
kubectl get deployment frontend -n devquote \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### **Logs**

```bash
# Backend
kubectl logs -f deployment/backend -n devquote

# Frontend
kubectl logs -f deployment/frontend -n devquote

# Argo CD
kubectl logs -n argocd deployment/argocd-application-controller
```

### **Pods**

```bash
# Listar
kubectl get pods -n devquote

# Watch (tempo real)
kubectl get pods -n devquote -w

# Descrever
kubectl describe pod POD_NAME -n devquote
```

### **Sync Manual (forçar)**

```bash
# Via kubectl
kubectl patch application devquote -n argocd \
  --type merge -p '{"operation":{"sync":{"revision":"HEAD"}}}'

# Via CLI (se instalado)
argocd app sync devquote --force
```

### **Testar Kustomize localmente**

```bash
# Renderizar manifests sem aplicar
kustomize build overlays/production/ | less

# Aplicar diretamente (sem Argo CD)
kubectl apply -k overlays/production/
```

---

## 📊 Recursos do Cluster

| Componente | Réplicas | RAM | CPU | Tag |
|------------|----------|-----|-----|-----|
| Backend | 1 | 512-750Mi | 400-800m | sha-XXXXXXX |
| Frontend | 1 | 128-256Mi | 100-200m | sha-XXXXXXX |
| PostgreSQL | 1 | 256-512Mi | 250-500m | 17 |
| Redis | 1 | 128-256Mi | 100-200m | 7 |
| Prometheus | 1 | 256-512Mi | 100-200m | latest |
| Grafana | 1 | 128-256Mi | 50-100m | latest |

---

## 🔐 Secrets

Ver documentação completa: [SECRETS.md](./SECRETS.md)

```bash
# Editar secrets localmente
vim k8s/secrets.yaml

# Criptografar
~/bin/kubeseal.exe --format=yaml \
  --cert=.sealed-secrets/public-key.pem \
  < k8s/secrets.yaml \
  > k8s/sealed-secrets.yaml

# Commitar apenas o criptografado
git add k8s/sealed-secrets.yaml
git commit -m "Update secrets"
git push
```

---

## 📚 Documentação

| Documento | Descrição |
|-----------|-----------|
| [KUSTOMIZE_MIGRATION.md](./KUSTOMIZE_MIGRATION.md) | Guia completo de migração e arquitetura |
| [ROLLBACK_GUIDE.md](./ROLLBACK_GUIDE.md) | Guia rápido de rollback (4 métodos) |
| [SECRETS.md](./SECRETS.md) | Gerenciamento de Sealed Secrets |
| [architecture.md](./architecture.md) | Visão geral da arquitetura |

---

## 🌐 Links

- **Aplicação:** https://devquote.com.br
- **Argo CD:** https://devquote.com.br/argocd
- **Grafana:** https://devquote.com.br/grafana
- **Swagger:** https://devquote.com.br/swagger-ui.html

---

## ✅ Status da Migração

- ✅ Estrutura Kustomize criada (base/ + overlays/)
- ✅ GitHub Actions atualizado (backend + frontend)
- ✅ Argo CD Application configurada
- ✅ Documentação completa
- ⏳ **Aguardando:** Teste e deploy

---

## 🎯 Próximos Passos

1. **Testar localmente:**
   ```bash
   kustomize build overlays/production/ | kubectl apply --dry-run=client -f -
   ```

2. **Aplicar Argo CD Application:**
   ```bash
   kubectl apply -f argocd/application.yaml
   ```

3. **Fazer primeiro deploy de teste:**
   ```bash
   cd devquote-backend
   echo "// test kustomize" >> README.md
   git add . && git commit -m "test: GitOps com Kustomize" && git push
   ```

4. **Acompanhar:**
   - GitHub Actions: https://github.com/wesleyeduardodev/devquote-backend/actions
   - Argo CD: https://devquote.com.br/argocd
   - Cluster: `kubectl get pods -n devquote -w`

5. **Testar rollback** (via Argo CD UI)

---

**Última atualização:** 2025-11-05
**Versão:** 2.0.0 (Kustomize + GitOps)
**Status:** ✅ Pronto para produção
