# Como Subir para o GitHub

Siga estes passos para criar o repositório no GitHub e fazer o primeiro push.

---

## 📋 Passo a Passo

### **1. Criar Repositório no GitHub**

1. Acesse: https://github.com/new
2. Preencha:
   - **Repository name**: `devquote-infra`
   - **Description**: "Kubernetes infrastructure for DevQuote application"
   - **Visibility**: ✅ Private (ou Public, você escolhe)
   - **⚠️ NÃO** marque "Initialize with README" (já temos README)
3. Clique em "Create repository"

---

### **2. Inicializar Git Local**

```bash
cd "C:/Users/Wesley Eduardo/Documents/projetos-git/devquote-infra"

# Inicializar git
git init

# Adicionar todos os arquivos
git add .

# Primeiro commit
git commit -m "feat: estrutura inicial do repositório de infraestrutura

- Manifestos Kubernetes organizados por componente
- Backend (2 réplicas), Frontend (1 réplica)
- PostgreSQL com persistência
- Prometheus + Grafana
- Ingress Traefik configurado
- Documentação completa"
```

---

### **3. Conectar ao GitHub**

```bash
# Adicionar remote (TROCAR pelo seu username)
git remote add origin https://github.com/wesleyeduardodev/devquote-infra.git

# Renomear branch para main
git branch -M main

# Push inicial
git push -u origin main
```

---

### **4. Verificar**

Acesse: `https://github.com/wesleyeduardodev/devquote-infra`

Você deverá ver:
- ✅ README.md renderizado
- ✅ Estrutura de pastas `k8s/`
- ✅ Documentação em `docs/`
- ✅ `.gitignore` protegendo secrets

---

## ⚠️ IMPORTANTE: Secrets

**ANTES de fazer o push**, certifique-se:

```bash
# Verificar se secrets.yaml está no gitignore
cat .gitignore | grep secrets.yaml

# Verificar status do git (secrets.yaml NÃO deve aparecer)
git status

# Se secrets.yaml aparecer, adicione ao .gitignore:
echo "k8s/secrets.yaml" >> .gitignore
git add .gitignore
git commit --amend --no-edit
```

---

## 🔄 Próximos Passos

Após subir o repositório:

1. ✅ Configurar Argo CD para monitorar este repo
2. ✅ Fazer push de mudanças
3. ✅ Argo CD aplica automaticamente no cluster

---

## 📝 Comandos Úteis (Futuro)

```bash
# Fazer mudanças
nano k8s/backend/deployment.yaml

# Commitar
git add k8s/backend/deployment.yaml
git commit -m "deploy: backend v1.2.3"
git push

# Argo CD detecta e aplica automaticamente! 🚀
```

---

## 🆘 Problemas?

### **Erro: repository not found**
- Verifique se o repositório foi criado no GitHub
- Verifique o nome do remote: `git remote -v`
- Troque pelo URL correto

### **Erro: authentication failed**
- Use GitHub token ao invés de senha
- Gere em: https://github.com/settings/tokens
- Use: `git push https://TOKEN@github.com/user/repo.git`

### **Secrets foram commitados por engano**
```bash
# Remover do histórico
git rm --cached k8s/secrets.yaml
git commit -m "chore: remover secrets do git"
git push --force

# ⚠️ TROCAR todas as senhas comprometidas!
```

---

## ✅ Checklist

- [ ] Repositório criado no GitHub
- [ ] Git inicializado localmente
- [ ] Primeiro commit criado
- [ ] Remote configurado
- [ ] Push realizado com sucesso
- [ ] Verificado no GitHub
- [ ] Secrets NÃO foram commitados
- [ ] README.md aparece renderizado

---

**Pronto! Repositório no ar!** 🎉
