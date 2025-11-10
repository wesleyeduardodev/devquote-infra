# Arquitetura DevQuote

## Stack Tecnológico

### Frontend
- React 18 + TypeScript + Tailwind CSS
- Nginx (produção)
- 1 réplica

### Backend
- Spring Boot 3.5.4 + Java 17
- JWT Authentication
- Integração AWS S3
- 2 réplicas (Alta Disponibilidade)

### Banco de Dados
- PostgreSQL 17
- StatefulSet com volume persistente

### Infraestrutura
- K3s (Kubernetes)
- Traefik (Ingress + Load Balancer)
- Argo CD (GitOps)
- Sealed Secrets (Gerenciamento de credenciais)

### Observabilidade
- Prometheus (Coleta de métricas)
- Grafana (Visualização)

---

## Fluxo de Requisição

```
Cliente (HTTPS)
      ↓
Traefik Ingress
      ↓
  ┌───────────────┐
  │ Path Matching │
  └───┬───────┬───┘
      │       │
   /api      /
      │       │
   Backend  Frontend
      │
  PostgreSQL
```

### Roteamento

| Path | Destino |
|------|---------|
| `/api` | Backend |
| `/actuator` | Backend (health/metrics) |
| `/grafana` | Grafana |
| `/argocd` | Argo CD |
| `/` | Frontend |

---

## Segurança

- HTTPS obrigatório (Let's Encrypt)
- JWT Authentication
- Sealed Secrets (RSA 4096-bit)
- Serviços internos não expostos (ClusterIP)
- Resource limits por pod

---

## Alta Disponibilidade

- Backend com 2 réplicas
- Health checks (liveness + readiness)
- Self-healing (restart automático)
- Rolling updates (zero downtime)

---

## CI/CD Pipeline

```
Developer
   ↓ git push
GitHub Actions
   ↓ build + test
Docker Hub
   ↓ update tag
devquote-infra
   ↓ git push
Argo CD
   ↓ auto-sync
K3s Cluster
   ↓ rolling update
Production
```

---

## Recursos

| Componente | Réplicas | RAM (Request/Limit) | CPU (Request/Limit) |
|------------|----------|---------------------|---------------------|
| PostgreSQL | 1 | 256Mi/512Mi | 250m/500m |
| Backend | 2 | 512Mi/750Mi | 300m/500m |
| Frontend | 1 | 128Mi/256Mi | 100m/200m |
| Prometheus | 1 | 256Mi/512Mi | 100m/200m |
| Grafana | 1 | 128Mi/256Mi | 50m/100m |
