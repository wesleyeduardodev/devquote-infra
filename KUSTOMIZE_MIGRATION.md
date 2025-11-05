# 🚀 Migração para Kustomize + GitOps Profissional

Este guia documenta a migração de tags `:latest` para **tags SHA imutáveis** usando **Kustomize**, seguindo as **melhores práticas da indústria**.

---

## 📋 O que mudou?

### ❌ Antes (Anti-pattern)

```yaml
# deployment.yaml
image: wesleyeduardodev/devquote-backend:latest
```

**Problemas:**
- ❌ Rollback impossível (tag `:latest` sempre aponta para versão mais recente)
- ❌ Não reproduzível
- ❌ Auditoria impossível
- ❌ Viola princípios GitOps

### ✅ Depois (Best Practice)

```yaml
# overlays/production/kustomization.yaml
images:
  - name: wesleyeduardodev/devquote-backend
    newTag: sha-a1b2c3d  # TAG IMUTÁVEL (SHA do commit)
```

**Vantagens:**
- ✅ Rollback funcional (via Argo CD ou Git)
- ✅ 100% reproduzível
- ✅ Rastreabilidade completa (SHA Git = SHA Docker)
- ✅ GitOps puro
- ✅ Histórico completo de deploys

---

## 🏗️ Nova Estrutura

```
devquote-infra/
├── base/                              # Configuração base (sem tags)
│   ├── kustomization.yaml
│   ├── backend/
│   │   ├── deployment.yaml            # Sem tag :latest
│   │   └── service.yaml
│   └── frontend/
│       ├── deployment.yaml            # Sem tag :latest
│       └── service.yaml
│
├── overlays/                          # Sobrescritas por ambiente
│   └── production/
│       └── kustomization.yaml         # ← Tags SHA definidas aqui
│
├── argocd/
│   └── application.yaml               # Configuração Argo CD
│
└── k8s/                               # Recursos compartilhados
    ├── namespace.yaml
    ├── sealed-secrets.yaml
    ├── database/
    ├── redis/
    ├── ingress/
    └── monitoring/
```

---

## 🔄 Fluxo de Deploy

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. DEVELOPER                                                    │
│    git push → devquote-backend repo                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. GITHUB ACTIONS (Backend CI/CD)                               │
│    ✓ Build Maven                                                │
│    ✓ Build Docker image → sha-a1b2c3d                           │
│    ✓ Push Docker Hub                                            │
│    ✓ Clone devquote-infra repo                                  │
│    ✓ kustomize edit set image ...sha-a1b2c3d                    │
│    ✓ git commit + push → devquote-infra                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. ARGO CD (GitOps)                                             │
│    ✓ Detecta commit no repo infra                              │
│    ✓ Diff: sha-xyz123 → sha-a1b2c3d                            │
│    ✓ Apply no cluster (rolling update)                         │
│    ✓ Health check automático                                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. KUBERNETES                                                   │
│    ✓ Backend: sha-a1b2c3d (PROD)                                │
│    ✓ Zero downtime (rolling update)                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📦 Passo a Passo: Migração

### **1. Configurar GitHub Personal Access Token (PAT)**

O CI/CD precisa de permissão para atualizar o repositório `devquote-infra`.

```bash
# 1. Criar PAT no GitHub:
# https://github.com/settings/tokens/new

# Permissões necessárias:
# ✓ repo (Full control)

# 2. Adicionar como secret no repositório BACKEND:
# Settings → Secrets → Actions → New repository secret
# Nome: GH_PAT
# Valor: ghp_xxxxxxxxxxxxxxxxxxxx

# 3. Adicionar como secret no repositório FRONTEND:
# (Mesmo processo, mesmo token)
```

### **2. Aplicar a Application do Argo CD**

```bash
# Aplicar configuração Argo CD
kubectl apply -f argocd/application.yaml

# Verificar se foi criada
kubectl get application -n argocd

# Ver status
kubectl describe application devquote -n argocd
```

### **3. Configurar Argo CD UI**

Acesse: https://devquote.com.br/argocd

Configurações recomendadas na UI:
- ✅ Auto-Sync: **Enabled**
- ✅ Self-Heal: **Enabled**
- ✅ Prune: **Enabled**

### **4. Primeiro Deploy (Teste)**

```bash
# 1. Fazer um commit simples no backend
cd devquote-backend
echo "// test" >> src/main/java/br/com/devquote/DevquoteApplication.java
git add .
git commit -m "test: Testar novo fluxo GitOps"
git push

# 2. Acompanhar no GitHub Actions
# https://github.com/wesleyeduardodev/devquote-backend/actions

# 3. Verificar commit no repo infra
cd ../devquote-infra
git pull
cat overlays/production/kustomization.yaml
# Deve ter: newTag: sha-XXXXXXX

# 4. Acompanhar no Argo CD
# https://devquote.com.br/argocd

# 5. Verificar no cluster
kubectl get pods -n devquote -w
kubectl describe pod backend-XXXXX -n devquote | grep Image:
```

---

## 🔙 Rollback

### **Opção 1: Via Argo CD UI (Recomendado)**

1. Acesse: https://devquote.com.br/argocd
2. Clique na aplicação **devquote**
3. Clique em **HISTORY AND ROLLBACK**
4. Selecione a revisão desejada
5. Clique em **ROLLBACK**

**Argo CD irá:**
- ✅ Reverter commit no Git
- ✅ Aplicar versão anterior da imagem
- ✅ Rolling update automático

### **Opção 2: Via Git (Manual)**

```bash
cd devquote-infra

# Ver histórico de commits
git log --oneline overlays/production/kustomization.yaml

# Exemplo de output:
# a1b2c3d 🚀 Update backend image to sha-xyz789
# def4567 🚀 Update backend image to sha-abc123  ← QUERO ESTA
# 8901234 🚀 Update backend image to sha-old999

# Reverter para commit específico
git revert a1b2c3d --no-commit
git commit -m "Rollback backend to sha-abc123"
git push

# Argo CD aplica automaticamente
```

### **Opção 3: Via Kustomize (Edição Manual)**

```bash
cd devquote-infra/overlays/production

# Editar manualmente
vim kustomization.yaml

# Alterar de:
# newTag: sha-xyz789

# Para:
# newTag: sha-abc123

# Commit e push
git add kustomization.yaml
git commit -m "Rollback backend to sha-abc123"
git push
```

### **Opção 4: Via kubectl (Emergência)**

Se Argo CD estiver fora ou precisar de rollback IMEDIATO:

```bash
# Ver histórico de revisões
kubectl rollout history deployment/backend -n devquote

# REVISION  CHANGE-CAUSE
# 1         Initial deploy
# 2         sha-abc123
# 3         sha-xyz789

# Rollback para revisão anterior
kubectl rollout undo deployment/backend -n devquote

# OU rollback para revisão específica
kubectl rollout undo deployment/backend -n devquote --to-revision=2

# ⚠️ IMPORTANTE: Sincronizar com Git depois!
# Caso contrário, Argo CD vai reverter de volta
```

---

## 🔍 Verificação e Troubleshooting

### **Ver versão atual em produção**

```bash
# Ver imagem do backend
kubectl get deployment backend -n devquote -o jsonpath='{.spec.template.spec.containers[0].image}'
# Output: wesleyeduardodev/devquote-backend:sha-a1b2c3d

# Ver imagem do frontend
kubectl get deployment frontend -n devquote -o jsonpath='{.spec.template.spec.containers[0].image}'
# Output: wesleyeduardodev/devquote-frontend:sha-x9y8z7

# Ver todas as versões
kubectl get pods -n devquote -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

### **Ver histórico completo de deploys**

```bash
# Via Git (fonte da verdade)
cd devquote-infra
git log --oneline --graph overlays/production/kustomization.yaml

# Via Kubernetes
kubectl rollout history deployment/backend -n devquote
kubectl rollout history deployment/frontend -n devquote

# Via Argo CD CLI
argocd app history devquote
```

### **Troubleshooting: Argo CD não sincroniza**

```bash
# Forçar sync manual
kubectl patch application devquote -n argocd --type merge -p '{"operation":{"sync":{"revision":"HEAD"}}}'

# Ou via CLI
argocd app sync devquote --force

# Ver logs do Argo CD
kubectl logs -n argocd deployment/argocd-application-controller
```

### **Troubleshooting: CI/CD falha ao atualizar infra**

```bash
# Verificar se GH_PAT tem permissões corretas
# Settings → Secrets → GH_PAT

# Testar clone manual
git clone https://GH_PAT@github.com/wesleyeduardodev/devquote-infra.git

# Ver logs do workflow
# GitHub Actions → Ver workflow que falhou → Ver step "Update Kubernetes manifests"
```

---

## 📊 Comparação: Antes vs Depois

| Característica | `:latest` (Antes) | SHA + Kustomize (Depois) |
|---------------|-------------------|--------------------------|
| **Rollback** | ❌ Impossível | ✅ 1 clique no Argo CD |
| **Rastreabilidade** | ❌ Não | ✅ SHA Git = SHA Docker |
| **Reproduzível** | ❌ Não | ✅ 100% |
| **Auditoria** | ❌ Não | ✅ Git log completo |
| **GitOps** | ❌ Parcial | ✅ Total |
| **Multi-ambiente** | ❌ Difícil | ✅ Overlays Kustomize |
| **Usado em Produção** | ⚠️ Apenas dev | ✅ Google, GitLab, CNCF |

---

## 🎯 Próximos Passos (Opcional)

### **1. Adicionar ambiente de Staging**

```bash
# Criar overlay de staging
mkdir -p overlays/staging
cp overlays/production/kustomization.yaml overlays/staging/

# Ajustar configs (menos recursos, etc)
vim overlays/staging/kustomization.yaml

# Criar Application Argo CD para staging
cp argocd/application.yaml argocd/application-staging.yaml
# Alterar: metadata.name, spec.source.path, spec.destination.namespace
```

### **2. Adicionar notificações**

```bash
# Slack notifications no Argo CD
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $SLACK_TOKEN
  template.app-deployed: |
    message: Application {{.app.metadata.name}} deployed to {{.app.spec.destination.namespace}}
  trigger.on-deployed: |
    - when: app.status.sync.status == 'Synced'
      send: [app-deployed]
EOF
```

### **3. Adicionar Health Checks customizados**

Ver: https://argo-cd.readthedocs.io/en/stable/operator-manual/health/

---

## 📚 Referências

- **Kustomize Docs:** https://kustomize.io/
- **Argo CD Docs:** https://argo-cd.readthedocs.io/
- **GitOps Principles:** https://opengitops.dev/
- **CNCF Best Practices:** https://www.cncf.io/blog/2021/05/25/gitops-is-the-path-forward/

---

## ✅ Checklist de Migração

- [ ] GH_PAT criado e configurado nos secrets (backend + frontend)
- [ ] Argo CD Application aplicada: `kubectl apply -f argocd/application.yaml`
- [ ] Argo CD UI configurado (auto-sync, self-heal, prune)
- [ ] Teste de deploy do backend (commit → GitHub Actions → Argo CD → K8s)
- [ ] Teste de deploy do frontend (commit → GitHub Actions → Argo CD → K8s)
- [ ] Teste de rollback via Argo CD UI
- [ ] Teste de rollback via Git revert
- [ ] Documentação lida e entendida
- [ ] Time treinado nos novos processos

---

**Última atualização:** 2025-11-05
**Versão:** 1.0.0
**Status:** ✅ Pronto para produção
