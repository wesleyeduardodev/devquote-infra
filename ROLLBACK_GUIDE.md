# 🔙 Guia Rápido de Rollback

## 🚨 Cenário: Backend com problema em produção

### **Método 1: Argo CD UI (Mais Rápido)** ⚡

```
1. Acesse: https://devquote.com.br/argocd
2. Clique em: devquote
3. Clique em: HISTORY AND ROLLBACK
4. Selecione: Revisão anterior (ou específica)
5. Clique em: ROLLBACK

⏱️ Tempo: ~30 segundos
✅ Argo CD aplica automaticamente no cluster
```

---

### **Método 2: Git Revert (Mais Seguro)** 🛡️

```bash
# 1. Ver histórico
cd devquote-infra
git log --oneline overlays/production/kustomization.yaml

# Output exemplo:
# a1b2c3d 🚀 Update backend to sha-xyz789 ← ATUAL (COM PROBLEMA)
# def4567 🚀 Update backend to sha-abc123 ← ÚLTIMA VERSÃO BOA

# 2. Reverter commit problemático
git revert a1b2c3d --no-commit

# 3. Commit + Push
git commit -m "Rollback backend to sha-abc123"
git push

# 4. Argo CD aplica automaticamente em ~10 segundos

⏱️ Tempo: ~1 minuto
✅ Mantém histórico completo no Git
✅ Auditável
```

---

### **Método 3: Edição Manual (Mais Controle)** 🎯

```bash
# 1. Editar kustomization.yaml
cd devquote-infra/overlays/production
vim kustomization.yaml

# 2. Mudar tag:
images:
  - name: wesleyeduardodev/devquote-backend
    newTag: sha-abc123  # ← Tag da versão estável

# 3. Commit + Push
git add kustomization.yaml
git commit -m "Rollback backend to sha-abc123"
git push

⏱️ Tempo: ~1-2 minutos
✅ Controle total sobre qual versão usar
```

---

### **Método 4: kubectl (Emergência SEM Git)** 🆘

```bash
# ⚠️ USE APENAS EM EMERGÊNCIAS
# (Argo CD pode reverter se auto-sync estiver ativo)

# Ver histórico de revisões
kubectl rollout history deployment/backend -n devquote

# Rollback para versão anterior
kubectl rollout undo deployment/backend -n devquote

# OU para revisão específica
kubectl rollout undo deployment/backend -n devquote --to-revision=2

# ⚠️ DEPOIS: Sincronizar com Git!
# Caso contrário, Argo CD vai reverter de volta

⏱️ Tempo: ~10 segundos
⚠️ NÃO mantém GitOps (temporário)
```

---

## 🔍 Verificar Versão Atual

```bash
# Backend
kubectl get deployment backend -n devquote \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Frontend
kubectl get deployment frontend -n devquote \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Ambos
kubectl get deployments -n devquote \
  -o custom-columns=NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image
```

---

## 📊 Ver Histórico de Versões

```bash
# Via Git (Fonte da verdade)
cd devquote-infra
git log --oneline --graph --all overlays/production/kustomization.yaml

# Via Kubernetes
kubectl rollout history deployment/backend -n devquote
kubectl rollout history deployment/frontend -n devquote

# Via Argo CD CLI (se instalado)
argocd app history devquote
```

---

## 🎯 Decisão Rápida: Qual método usar?

| Situação | Método Recomendado | Tempo |
|----------|-------------------|-------|
| 🔥 Produção quebrada | **Argo CD UI** | 30s |
| ✅ Rollback controlado | **Git Revert** | 1min |
| 🎯 Versão específica | **Edição Manual** | 2min |
| 🆘 Argo CD fora | **kubectl** | 10s |

---

## ⚠️ Importante: Após rollback manual (kubectl)

Se você usou `kubectl rollout undo`:

```bash
# 1. Ver qual tag está rodando agora
kubectl get deployment backend -n devquote \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# Output: wesleyeduardodev/devquote-backend:sha-abc123

# 2. Atualizar Git para refletir o estado atual
cd devquote-infra/overlays/production
vim kustomization.yaml
# Alterar newTag para: sha-abc123

# 3. Commit
git add kustomization.yaml
git commit -m "Sync Git with emergency rollback (sha-abc123)"
git push
```

---

## 🧪 Testar Rollback (Ambiente Seguro)

Antes de precisar fazer rollback em produção, teste o processo:

```bash
# 1. Fazer 2 deploys consecutivos do backend
cd devquote-backend
echo "// deploy 1" >> README.md && git add . && git commit -m "test 1" && git push
# Aguardar deploy...

echo "// deploy 2" >> README.md && git add . && git commit -m "test 2" && git push
# Aguardar deploy...

# 2. Ver histórico no Argo CD
# https://devquote.com.br/argocd → devquote → HISTORY

# 3. Fazer rollback para "test 1"
# HISTORY AND ROLLBACK → Selecionar → ROLLBACK

# 4. Confirmar
kubectl get pods -n devquote -w
```

---

## 📞 Contatos de Emergência

- **Argo CD:** https://devquote.com.br/argocd
- **Grafana:** https://devquote.com.br/grafana
- **GitHub Backend:** https://github.com/wesleyeduardodev/devquote-backend
- **GitHub Frontend:** https://github.com/wesleyeduardodev/devquote-frontend
- **GitHub Infra:** https://github.com/wesleyeduardodev/devquote-infra

---

**Última atualização:** 2025-11-05
