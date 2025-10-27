# DevQuote Infrastructure

Repositório de infraestrutura do **DevQuote** - Manifestos Kubernetes (K3s) para deploy GitOps com Argo CD.

## 📋 Visão Geral

Este repositório contém todos os manifestos Kubernetes necessários para executar a aplicação DevQuote em produção usando:

- **K3s** (Kubernetes leve)
- **Traefik** (Ingress Controller)
- **PostgreSQL 17** (Banco de dados)
- **Prometheus + Grafana** (Observabilidade)
- **Argo CD** (GitOps)

---

## 🏗️ Arquitetura

```
Internet (HTTPS)
      ↓
Traefik Ingress
      ↓
   ┌──────────────────────┐
   │   devquote.com.br    │
   └──────────┬───────────┘
              │
   ┌──────────┴────────────┐
   │                       │
Frontend (1 pod)      Backend (2 pods)
                           │
                    PostgreSQL (1 pod)
```

---

## 📁 Estrutura do Repositório

```
devquote-infra/
├── README.md                     # Este arquivo
├── .gitignore                    # Protege secrets
│
├── k8s/                          # Manifestos Kubernetes
│   ├── namespace.yaml            # Namespace devquote
│   ├── secrets.yaml              # ⚠️ NÃO commitado (use .example)
│   ├── secrets.yaml.example      # Template dos secrets
│   │
│   ├── backend/                  # Backend (Spring Boot)
│   │   ├── deployment.yaml       # 2 réplicas, 750Mi RAM
│   │   └── service.yaml
│   │
│   ├── frontend/                 # Frontend (React)
│   │   ├── deployment.yaml       # 1 réplica, 256Mi RAM
│   │   └── service.yaml
│   │
│   ├── database/                 # PostgreSQL 17
│   │   ├── statefulset.yaml      # StatefulSet persistente
│   │   ├── service.yaml
│   │   └── pvc.yaml              # 10Gi storage
│   │
│   ├── ingress/                  # Traefik Ingress
│   │   └── ingress.yaml          # HTTPS + routing
│   │
│   └── monitoring/               # Observabilidade
│       ├── prometheus/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   ├── configmap.yaml
│       │   └── pvc.yaml          # 5Gi storage
│       └── grafana/
│           ├── deployment.yaml
│           ├── service.yaml
│           └── pvc.yaml          # 2Gi storage
│
└── docs/                         # Documentação
    ├── architecture.md
    └── deployment.md
```

---

## 🚀 Deployment

### **Pré-requisitos**

- ✅ Cluster K3s instalado
- ✅ `kubectl` configurado
- ✅ Certificados SSL (Let's Encrypt)
- ✅ Docker Hub com imagens: `backend:latest` e `frontend:latest`

### **1. Configurar Secrets**

```bash
# Copiar template
cp k8s/secrets.yaml.example k8s/secrets.yaml

# Editar com valores reais
nano k8s/secrets.yaml

# Criar secret TLS (certificado SSL)
kubectl create secret tls devquote-tls \
  --cert=/etc/letsencrypt/live/devquote.com.br/fullchain.pem \
  --key=/etc/letsencrypt/live/devquote.com.br/privkey.pem \
  -n devquote
```

### **2. Aplicar Manifestos**

```bash
# Aplicar em ordem
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secrets.yaml

# Database
kubectl apply -f k8s/database/

# Backend e Frontend
kubectl apply -f k8s/backend/
kubectl apply -f k8s/frontend/

# Ingress
kubectl apply -f k8s/ingress/

# Monitoring
kubectl apply -f k8s/monitoring/prometheus/
kubectl apply -f k8s/monitoring/grafana/
```

### **3. Verificar Status**

```bash
# Ver pods
kubectl get pods -n devquote

# Ver serviços
kubectl get svc -n devquote

# Ver ingress
kubectl get ingress -n devquote

# Logs do backend
kubectl logs -f deployment/backend -n devquote
```

---

## 🔄 GitOps com Argo CD

### **Workflow Completo**

1. **Dev faz mudança** no código → Push para `devquote-backend` ou `devquote-frontend`
2. **GitHub Actions** roda:
   - Build com Maven/NPM
   - Build da imagem Docker
   - Push para Docker Hub com nova tag (ex: `v1.2.3`)
3. **Dev atualiza** este repo (`devquote-infra`):
   - Edita `k8s/backend/deployment.yaml` → Muda tag da imagem
   - Commit e push
4. **Argo CD detecta** mudança no Git (automático)
5. **Argo CD aplica** no cluster K3s
6. **Rolling update** sem downtime! 🎉

### **Exemplo de Atualização**

```bash
# 1. Backend foi buildado e gerou v1.2.3

# 2. Atualizar manifesto
cd devquote-infra
nano k8s/backend/deployment.yaml

# Trocar:
# image: wesleyeduardodev/devquote-backend:latest
# Por:
# image: wesleyeduardodev/devquote-backend:v1.2.3

# 3. Commit
git add k8s/backend/deployment.yaml
git commit -m "deploy: backend v1.2.3"
git push

# 4. Argo CD detecta e aplica automaticamente! 🚀
```

---

## 📊 Recursos

| Componente | Réplicas | RAM Request | RAM Limit | CPU Request | CPU Limit |
|------------|----------|-------------|-----------|-------------|-----------|
| PostgreSQL | 1 | 256Mi | 512Mi | 250m | 500m |
| Backend | 2 | 512Mi | 750Mi | 300m | 500m |
| Frontend | 1 | 128Mi | 256Mi | 100m | 200m |
| Prometheus | 1 | 256Mi | 512Mi | 100m | 200m |
| Grafana | 1 | 128Mi | 256Mi | 50m | 100m |
| **Total** | **6 pods** | **1.76Gi** | **3.29Gi** | **1150m** | **2000m** |

**VPS:** 8GB RAM, 2 CPUs → Uso estimado: **40-50% RAM**

---

## 🔐 Segurança

### **Secrets**

- ⚠️ **NUNCA** commite `k8s/secrets.yaml`
- ✅ Use `k8s/secrets.yaml.example` como template
- ✅ `.gitignore` protege secrets automaticamente

### **TLS**

- ✅ HTTPS obrigatório via Traefik
- ✅ Certificados Let's Encrypt
- ✅ Renovação automática (certbot + cron)

### **Access Control**

- ✅ Backend expõe apenas `/api`, `/actuator`, `/oauth2`
- ✅ Frontend serve arquivos estáticos
- ✅ PostgreSQL apenas interno (ClusterIP)

---

## 🛠️ Comandos Úteis

```bash
# Ver uso de recursos
kubectl top nodes
kubectl top pods -n devquote

# Logs em tempo real
kubectl logs -f deployment/backend -n devquote
kubectl logs -f deployment/frontend -n devquote

# Escalar backend
kubectl scale deployment backend --replicas=3 -n devquote

# Restart de um deployment
kubectl rollout restart deployment/backend -n devquote

# Ver histórico de deploys
kubectl rollout history deployment/backend -n devquote

# Rollback
kubectl rollout undo deployment/backend -n devquote

# Describe (debug)
kubectl describe pod <pod-name> -n devquote

# Entrar em um pod
kubectl exec -it <pod-name> -n devquote -- /bin/sh
```

---

## 📈 Observabilidade

### **Prometheus**

- **URL interna**: `http://prometheus-service:9090`
- **Scrape**: Backend `/actuator/prometheus` a cada 15s
- **Retenção**: 15 dias

### **Grafana**

- **URL**: `https://devquote.com.br/grafana`
- **Usuário**: `admin`
- **Senha**: `admin123` (⚠️ Trocar em produção!)

---

## 🔄 Rollback

Se algo der errado após um deploy:

```bash
# Ver histórico
kubectl rollout history deployment/backend -n devquote

# Rollback para versão anterior
kubectl rollout undo deployment/backend -n devquote

# Rollback para versão específica
kubectl rollout undo deployment/backend --to-revision=2 -n devquote
```

Ou via Git:
```bash
git revert HEAD
git push
# Argo CD aplica automaticamente
```

---

## 📝 Links

- **Aplicação**: https://devquote.com.br
- **Grafana**: https://devquote.com.br/grafana
- **Backend Health**: https://devquote.com.br/actuator/health
- **Repositório Backend**: https://github.com/wesleyeduardodev/devquote-backend
- **Repositório Frontend**: https://github.com/wesleyeduardodev/devquote-frontend

---

## 🤝 Contribuindo

1. Faça mudanças nos manifestos
2. Teste localmente com `kubectl apply --dry-run=client`
3. Commit com mensagem descritiva
4. Push para `main`
5. Argo CD aplica automaticamente

---

## 📄 Licença

Propriedade de DevQuote - Todos os direitos reservados.

---

## 👤 Autor

**Wesley Eduardo**
- GitHub: [@wesleyeduardodev](https://github.com/wesleyeduardodev)
- Email: wesleyeduardo.dev@gmail.com
