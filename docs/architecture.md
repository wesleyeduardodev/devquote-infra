# Arquitetura DevQuote

## 📊 Visão Geral

DevQuote utiliza uma arquitetura moderna baseada em microserviços, containerizada com Docker e orquestrada por Kubernetes (K3s).

---

## 🏗️ Componentes

### **1. Frontend (React + Vite)**
- **Tecnologia**: React 18, TypeScript, Tailwind CSS
- **Servidor**: Nginx
- **Réplicas**: 1
- **Build Size**: ~716KB
- **Responsabilidades**:
  - UI/UX da aplicação
  - Autenticação OAuth2
  - Dashboard e visualizações
  - Consumo de APIs REST

### **2. Backend (Spring Boot)**
- **Tecnologia**: Spring Boot 3.5.4, Java 21
- **Réplicas**: 2 (Alta Disponibilidade)
- **Features**:
  - API RESTful
  - OAuth2 + JWT
  - Integração AWS S3
  - Email transacional
  - Métricas Prometheus
- **Endpoints Principais**:
  - `/api/*` - APIs de negócio
  - `/oauth2/*` - Autenticação
  - `/actuator/*` - Health checks e métricas

### **3. Banco de Dados (PostgreSQL 17)**
- **Tipo**: StatefulSet (persistente)
- **Storage**: 10GB PersistentVolume
- **Backup**: Diário via cron → S3
- **Retenção**: 7 dias local, ilimitado S3

### **4. Ingress (Traefik)**
- **Função**: Reverse proxy e load balancer
- **Features**:
  - Terminação TLS/SSL
  - Roteamento baseado em path
  - Renovação automática de certificados
- **Regras**:
  - `/api` → Backend
  - `/actuator` → Backend
  - `/oauth2` → Backend
  - `/` → Frontend

### **5. Observabilidade**

#### **Prometheus**
- Coleta de métricas do backend
- Scrape a cada 15s
- Retenção: 15 dias
- Storage: 5GB

#### **Grafana**
- Visualização de métricas
- Dashboards customizados
- Storage: 2GB
- URL: `/grafana`

---

## 🔄 Fluxo de Requisição

```
Cliente (Browser)
      ↓ HTTPS
Traefik Ingress (Port 443)
      ↓
  ┌───────────────┐
  │ Path Matching │
  └───┬───────┬───┘
      │       │
      ↓       ↓
   /api     /
      │       │
   Backend  Frontend
      │
      ↓
  PostgreSQL
```

### **Exemplo 1: Login**
```
1. User acessa https://devquote.com.br
2. Traefik → Frontend pod
3. Frontend exibe tela de login
4. User clica "Login com Google"
5. Frontend → Backend (/oauth2/authorization/google)
6. Backend redireciona para Google OAuth
7. Google autentica e retorna para backend
8. Backend gera JWT e retorna para frontend
9. Frontend armazena token e redireciona para dashboard
```

### **Exemplo 2: Listar Projetos**
```
1. Frontend faz GET /api/projects (com JWT no header)
2. Traefik identifica /api → Backend pod 1 ou 2 (load balancing)
3. Backend valida JWT
4. Backend consulta PostgreSQL
5. Backend retorna JSON
6. Frontend renderiza lista
```

---

## 🔐 Segurança

### **Camadas de Segurança**

1. **Rede**:
   - Firewall (UFW): Apenas portas 22, 80, 443, 6443
   - ClusterIP: Serviços internos não expostos

2. **Aplicação**:
   - OAuth2 + JWT
   - CORS configurado
   - Headers de segurança (HSTS, X-Frame-Options)

3. **Dados**:
   - Secrets Kubernetes (base64)
   - TLS em trânsito
   - Backup criptografado S3

4. **Infraestrutura**:
   - SSH key-based apenas
   - K3s com RBAC
   - Resource limits por pod

---

## 📈 Escalabilidade

### **Atual**
- Backend: 2 réplicas
- Frontend: 1 réplica
- PostgreSQL: 1 instância

### **Como Escalar**

#### **Horizontal (mais pods)**
```bash
# Backend
kubectl scale deployment backend --replicas=3 -n devquote

# Frontend
kubectl scale deployment frontend --replicas=2 -n devquote
```

#### **Vertical (mais recursos)**
Editar `k8s/backend/deployment.yaml`:
```yaml
resources:
  requests:
    memory: "1Gi"  # Era 512Mi
    cpu: "500m"    # Era 300m
  limits:
    memory: "1.5Gi"  # Era 750Mi
    cpu: "1000m"     # Era 500m
```

### **Limites do VPS Atual**
- 8GB RAM total
- ~2.5GB usados atualmente (31%)
- Pode suportar até:
  - Backend: 4-5 réplicas
  - Frontend: 3-4 réplicas

---

## 🔄 CI/CD Pipeline

```
Developer
   ↓ (git push)
GitHub
   ↓ (webhook)
GitHub Actions
   ↓ (build)
Docker Image
   ↓ (push)
Docker Hub
   ↓ (tag update)
devquote-infra repo
   ↓ (git push)
Argo CD
   ↓ (sync)
K3s Cluster
   ↓ (rolling update)
Production! 🎉
```

### **Benefícios**
- ✅ Zero downtime deploys
- ✅ Rollback automático se falhar health check
- ✅ Git como fonte da verdade
- ✅ Auditoria completa via commits

---

## 🛡️ Alta Disponibilidade

### **Estratégias Implementadas**

1. **Backend (2 réplicas)**
   - Se 1 pod cair, o outro continua
   - Load balancing automático
   - Health checks (liveness + readiness)

2. **Self-Healing**
   - K3s monitora pods continuamente
   - Se um pod crashar, recria automaticamente
   - Restart automático se health check falhar

3. **Rolling Updates**
   - Deploy gradual: 1 pod por vez
   - Aguarda readiness antes de continuar
   - Rollback automático se falhar

### **SLA Esperado**
- **Uptime**: ~99.5% (considerando VPS single point)
- **Downtime planejado**: 0 (rolling updates)
- **Recovery time**: ~30 segundos (pod restart)

---

## 📊 Performance

### **Métricas Atuais**

| Métrica | Backend | Frontend | PostgreSQL |
|---------|---------|----------|------------|
| **CPU** | 15-20m | 1m | 3m |
| **RAM** | 370-400Mi | 3Mi | 61Mi |
| **Startup** | ~60s | ~5s | ~10s |
| **Response Time** | ~100ms | ~10ms | N/A |

### **Otimizações Aplicadas**

1. **Backend**:
   - Connection pooling (HikariCP)
   - JPA cache
   - Imagem Alpine (menor)

2. **Frontend**:
   - Build otimizado (Vite)
   - Compressão Gzip (Nginx)
   - Cache de assets

3. **PostgreSQL**:
   - Shared buffers otimizados
   - Max connections limitado
   - Logs minimizados

---

## 🔮 Roadmap

### **Fase 4 - Argo CD** (Em andamento)
- [ ] Instalar Argo CD
- [ ] Configurar GitOps
- [ ] Auto-sync

### **Fase 5 - Melhorias**
- [ ] Adicionar Redis cache
- [ ] Implementar rate limiting
- [ ] CDN para assets estáticos
- [ ] Multi-region (futuro)

---

## 📚 Referências

- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [12 Factor App](https://12factor.net/)
- [GitOps Principles](https://opengitops.dev/)
