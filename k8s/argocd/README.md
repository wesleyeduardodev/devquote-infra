# 📧 Notificações Argo CD

Sistema de notificações por email para eventos do Argo CD (sync failures, degraded apps, etc).

---

## 📋 O que notifica

### 1️⃣ **Sync Failed** (Falha na sincronização)
- Quando o Argo CD não consegue aplicar mudanças do Git
- Erros de YAML inválido
- Problemas de conectividade com cluster

### 2️⃣ **App Degraded** (Aplicação degradada)
- Pods crashando
- Readiness probes falhando
- Recursos sem as réplicas esperadas

### 3️⃣ **Out of Sync** (Fora de sincronia) - INFORMATIVO
- Git e cluster estão diferentes
- Auto-sync vai corrigir automaticamente

---

## 🚀 Como configurar

### 1. Criar Secret com credenciais SMTP

**Opção A: Manualmente (desenvolvimento)**
```bash
# Copiar template
cp argocd-notifications-secret.yaml.template argocd-notifications-secret.yaml

# Editar e preencher credenciais
nano argocd-notifications-secret.yaml

# Aplicar
kubectl apply -f argocd-notifications-secret.yaml

# IMPORTANTE: Adicionar no .gitignore
echo "argocd-notifications-secret.yaml" >> .gitignore
```

**Opção B: Sealed Secrets (produção - recomendado)**
```bash
# Criar secret temporário
kubectl create secret generic argocd-notifications-secret \
  --from-literal=email-username="seu.email@gmail.com" \
  --from-literal=email-password="sua-senha-app" \
  --namespace=argocd \
  --dry-run=client -o yaml > /tmp/argocd-notifications-secret.yaml

# Criptografar com Sealed Secrets
kubeseal --format=yaml \
  --cert=<(kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml) \
  < /tmp/argocd-notifications-secret.yaml \
  > argocd-notifications-sealed-secret.yaml

# Aplicar sealed secret
kubectl apply -f argocd-notifications-sealed-secret.yaml

# Remover arquivo temporário
rm /tmp/argocd-notifications-secret.yaml
```

---

### 2. Aplicar ConfigMap de notificações

```bash
kubectl apply -f argocd-notifications-cm.yaml
```

---

### 3. Reiniciar Argo CD Notifications Controller

```bash
kubectl rollout restart deployment argocd-notifications-controller -n argocd
```

Se o deployment não existir, instalar o controller:
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-notifications/release-1.0/manifests/install.yaml
```

---

## 🔍 Verificar se está funcionando

### Ver logs do controller
```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-notifications-controller -f
```

### Testar notificação manualmente
```bash
kubectl patch app devquote -n argocd --type json \
  -p='[{"op": "add", "path": "/metadata/annotations/notifications.argoproj.io~1test", "value": "true"}]'
```

---

## 📧 Emails que você receberá

**Para:** `wesleyeduardo.dev@gmail.com`

**Assuntos:**
- `[DevQuote] ❌ Argo CD Sync FALHOU - devquote`
- `[DevQuote] ⚠️ Aplicação DEGRADADA - devquote`
- `[DevQuote] 🔄 Aplicação fora de sincronia - devquote`

---

## 🔧 Customização

### Alterar email de destino

Editar `argocd-notifications-cm.yaml` linha 71:
```yaml
subscriptions: |
  - recipients:
    - seu.novo.email@exemplo.com  # ← Alterar aqui
```

### Adicionar mais triggers

Editar `argocd-notifications-cm.yaml` na seção `subscriptions`:
```yaml
triggers:
- on-sync-failed
- on-degraded
- on-out-of-sync  # ← Adicionar este para receber notificação de OutOfSync
```

---

## 📌 Requisitos

- ✅ Argo CD instalado no cluster
- ✅ Argo CD Notifications Controller (instalar se não tiver)
- ✅ Credenciais SMTP válidas (Gmail App Password recomendado)
- ✅ Porta 587 (STARTTLS) aberta no firewall

---

## ❓ Troubleshooting

### Não recebo emails

1. **Verificar se controller está rodando:**
```bash
kubectl get pods -n argocd | grep notifications
```

2. **Verificar logs:**
```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-notifications-controller
```

3. **Verificar secret:**
```bash
kubectl get secret argocd-notifications-secret -n argocd
```

4. **Testar credenciais SMTP:**
```bash
kubectl run test-email --rm -it --image=python:3.9-alpine --restart=Never -- python3 -c "
import smtplib
server = smtplib.SMTP('smtp.gmail.com', 587)
server.starttls()
server.login('seu.email@gmail.com', 'sua-senha-app')
print('✓ Credenciais válidas!')
server.quit()
"
```

---

### Gmail bloqueia envio

**Solução:** Usar **App Password** do Gmail (não a senha normal)

1. Acessar: https://myaccount.google.com/apppasswords
2. Criar nova senha de app para "Mail"
3. Usar essa senha no secret `email-password`

---

**Última atualização:** 2025-11-10
