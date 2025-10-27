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

## Referências

- [Sealed Secrets GitHub](https://github.com/bitnami-labs/sealed-secrets)
- [Documentação Oficial](https://sealed-secrets.netlify.app/)
