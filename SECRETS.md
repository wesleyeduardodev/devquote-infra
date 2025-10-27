# 🔐 Gerenciamento de Secrets - DevQuote

Este documento explica como gerenciar secrets (credenciais, senhas, API keys) no projeto DevQuote usando **Sealed Secrets**.

## 📋 Índice

- [Como Funciona](#como-funciona)
- [Arquivos Importantes](#arquivos-importantes)
- [Operações Comuns](#operações-comuns)
  - [Alterar uma Secret Existente](#alterar-uma-secret-existente)
  - [Adicionar Nova Secret](#adicionar-nova-secret)
  - [Remover uma Secret](#remover-uma-secret)
- [Troubleshooting](#troubleshooting)

---

## 🎯 Como Funciona

### Fluxo de Secrets

```
┌─────────────────────────────────────────────────────────────┐
│  1. DESENVOLVIMENTO (Seu PC)                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  secrets.yaml (texto claro)                                 │
│    POSTGRES_PASSWORD: MinhaSenh@123                         │
│         ↓                                                   │
│    kubeseal (criptografa)                                   │
│         ↓                                                   │
│  sealed-secrets.yaml (criptografado)                        │
│    POSTGRES_PASSWORD: AgBY8f7kx... (hash de 2048 chars)     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                         ↓
                    git commit
                    git push
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  2. REPOSITÓRIO GIT                                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  GitHub: devquote-infra                                     │
│    ✅ k8s/sealed-secrets.yaml (SEGURO - criptografado)      │
│    ❌ k8s/secrets.yaml (NÃO VAI PRO GIT - .gitignore)       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                         ↓
                     Argo CD
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  3. CLUSTER KUBERNETES                                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Argo CD aplica sealed-secrets.yaml                         │
│         ↓                                                   │
│  Sealed Secrets Controller (descriptografa)                 │
│         ↓                                                   │
│  Kubernetes Secret (texto claro, apenas dentro do cluster)  │
│         ↓                                                   │
│  Pods (leem as secrets como variáveis de ambiente)          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📂 Arquivos Importantes

```
devquote-infra/
├── .sealed-secrets/
│   └── public-key.pem              ← Chave pública (local, não vai pro Git)
├── k8s/
│   ├── secrets.yaml                ← TEXTO CLARO (nunca commitar!)
│   └── sealed-secrets.yaml         ← CRIPTOGRAFADO (vai pro Git ✅)
└── .gitignore
    ├── k8s/secrets.yaml            ← Protegido
    └── .sealed-secrets/            ← Protegido
```

### ⚠️ IMPORTANTE

- ✅ **SEMPRE commitar:** `k8s/sealed-secrets.yaml` (criptografado)
- ❌ **NUNCA commitar:** `k8s/secrets.yaml` (texto claro)

---

## 🛠️ Operações Comuns

### Alterar uma Secret Existente

**Exemplo:** Trocar a senha do PostgreSQL

#### Passo 1: Editar o arquivo local

```bash
cd devquote-infra

# Editar secrets.yaml (texto claro)
vim k8s/secrets.yaml
```

Altere a variável desejada:

```yaml
# DE:
POSTGRES_PASSWORD: SenhaAntiga123

# PARA:
POSTGRES_PASSWORD: NovaSenha@2025!SuperForte
```

#### Passo 2: Re-criptografar

```bash
# Criptografar com kubeseal
~/bin/kubeseal.exe --format=yaml \
  --cert=.sealed-secrets/public-key.pem \
  < k8s/secrets.yaml \
  > k8s/sealed-secrets.yaml

# Verificar que foi gerado
ls -lh k8s/sealed-secrets.yaml
```

#### Passo 3: Commitar e fazer push

```bash
# Adicionar ao Git
git add k8s/sealed-secrets.yaml

# Commitar
git commit -m "Update PostgreSQL password"

# Push para o GitHub
git push origin main
```

#### Passo 4: Aguardar Argo CD aplicar (automático)

O Argo CD detecta a mudança em 1-3 minutos e aplica automaticamente.

Para verificar:

```bash
# Ver status do Argo CD
kubectl get application devquote -n argocd

# Ver logs do Sealed Secrets Controller
kubectl logs -n kube-system -l app.kubernetes.io/name=sealed-secrets --tail=20
```

#### Passo 5: Reiniciar pods (se necessário)

```bash
# Reiniciar backend para usar nova senha
kubectl rollout restart deployment/backend -n devquote

# Verificar status
kubectl get pods -n devquote -l app=backend
```

---

### Adicionar Nova Secret

**Exemplo:** Adicionar API key do SendGrid

#### Passo 1: Editar secrets.yaml

```bash
vim k8s/secrets.yaml
```

Adicionar nova variável:

```yaml
stringData:
  # ... variáveis existentes ...
  SENDGRID_API_KEY: SG.abc123xyz789...     # ← NOVA
```

#### Passo 2: Re-criptografar

```bash
~/bin/kubeseal.exe --format=yaml \
  --cert=.sealed-secrets/public-key.pem \
  < k8s/secrets.yaml \
  > k8s/sealed-secrets.yaml
```

#### Passo 3: Commitar

```bash
git add k8s/sealed-secrets.yaml
git commit -m "Add SendGrid API key"
git push origin main
```

#### Passo 4: Aguardar Argo CD

Aguardar 1-3 minutos para aplicação automática.

#### Passo 5: Atualizar código da aplicação

Não esqueça de atualizar o código do backend/frontend para ler a nova variável:

**Backend (application.properties):**
```properties
sendgrid.api-key=${SENDGRID_API_KEY}
```

**Deployment (k8s/backend/deployment.yaml):**
Já está configurado para ler todas as variáveis do secret `devquote-secrets`.

---

### Remover uma Secret

**Exemplo:** Remover variável MAIL_PASSWORD (não usa mais Gmail)

#### Passo 1: Editar secrets.yaml

```bash
vim k8s/secrets.yaml
```

Remover a linha:

```yaml
stringData:
  # ... outras variáveis ...
  # MAIL_PASSWORD: bmgjzyoatrtxnmkd     ← REMOVIDO
```

#### Passo 2: Re-criptografar

```bash
~/bin/kubeseal.exe --format=yaml \
  --cert=.sealed-secrets/public-key.pem \
  < k8s/secrets.yaml \
  > k8s/sealed-secrets.yaml
```

#### Passo 3: Commitar

```bash
git add k8s/sealed-secrets.yaml
git commit -m "Remove MAIL_PASSWORD (migrated to SendGrid)"
git push origin main
```

#### Passo 4: Aguardar Argo CD

Aguardar aplicação automática.

#### Passo 5: Remover código que usa a variável

Atualizar backend para não depender mais de `MAIL_PASSWORD`.

---

## 🔧 Troubleshooting

### Erro: "error: unable to recognize STDIN: no matches for kind \"SealedSecret\""

**Problema:** Sealed Secrets Controller não instalado.

**Solução:**

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.2/controller.yaml

# Verificar instalação
kubectl get pods -n kube-system | grep sealed-secrets
```

---

### Erro: "cannot fetch certificate"

**Problema:** Chave pública não encontrada ou controller offline.

**Solução:**

```bash
# Buscar nova chave pública
~/bin/kubeseal.exe --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  > .sealed-secrets/public-key.pem

# Verificar controller
kubectl get pods -n kube-system -l app.kubernetes.io/name=sealed-secrets
```

---

### Secret não está sendo atualizado no cluster

**Problema:** Argo CD não sincronizou ou SealedSecret com problema.

**Solução:**

```bash
# Verificar status do Argo CD
kubectl get application devquote -n argocd

# Forçar sincronização
kubectl patch application devquote -n argocd \
  --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"main"}}}'

# Verificar SealedSecret
kubectl get sealedsecret -n devquote

# Verificar Secret gerado
kubectl get secret devquote-secrets -n devquote

# Ver logs do controller
kubectl logs -n kube-system -l app.kubernetes.io/name=sealed-secrets --tail=50
```

---

### Pods não estão lendo a nova secret

**Problema:** Pods foram criados antes da secret ser atualizada.

**Solução:**

```bash
# Reiniciar deployment
kubectl rollout restart deployment/backend -n devquote
kubectl rollout restart deployment/frontend -n devquote

# Verificar status
kubectl rollout status deployment/backend -n devquote
```

---

### Como verificar o valor de uma secret (descriptografado)?

**⚠️ CUIDADO:** Secrets contêm dados sensíveis!

```bash
# Ver todas as chaves
kubectl get secret devquote-secrets -n devquote -o jsonpath='{.data}' | grep -o '"[^"]*":'

# Ver valor específico (base64 encoded)
kubectl get secret devquote-secrets -n devquote -o jsonpath='{.data.POSTGRES_PASSWORD}'

# Descriptografar valor (USE COM CUIDADO!)
kubectl get secret devquote-secrets -n devquote -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d
```

---

## 🔒 Boas Práticas de Segurança

### ✅ Faça

- ✅ Sempre edite `k8s/secrets.yaml` localmente
- ✅ Sempre re-criptografe após mudanças
- ✅ Commit apenas `k8s/sealed-secrets.yaml`
- ✅ Rotacione senhas periodicamente
- ✅ Use senhas fortes (mínimo 16 caracteres)
- ✅ Documente mudanças nas mensagens de commit

### ❌ Não Faça

- ❌ NUNCA commitar `k8s/secrets.yaml` (texto claro)
- ❌ NUNCA compartilhar senhas em Slack/email
- ❌ NUNCA usar senhas fracas em produção
- ❌ NUNCA commitar chaves AWS, tokens, etc em código
- ❌ NUNCA expor secrets em logs

---

## 📚 Referências

- **Sealed Secrets GitHub:** https://github.com/bitnami-labs/sealed-secrets
- **Documentação Oficial:** https://sealed-secrets.netlify.app/
- **Argo CD Docs:** https://argo-cd.readthedocs.io/

---

## 🆘 Suporte

Em caso de dúvidas ou problemas:

1. Verificar logs do Sealed Secrets Controller
2. Verificar status do Argo CD
3. Verificar `.gitignore` (secrets.yaml deve estar listado)
4. Verificar se a chave pública está correta

**Comando útil para debug:**

```bash
# Ver todo o fluxo
kubectl get application devquote -n argocd && \
kubectl get sealedsecret -n devquote && \
kubectl get secret devquote-secrets -n devquote && \
kubectl get pods -n devquote
```
