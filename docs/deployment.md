# Guia de Deployment - DevQuote

Guia completo para fazer deploy da aplicação DevQuote no K3s.

---

## 🎯 Pré-requisitos

### **1. VPS Configurado**
- ✅ Ubuntu 24.04 LTS
- ✅ 8GB RAM, 2 CPUs
- ✅ K3s instalado
- ✅ kubectl configurado localmente
- ✅ Firewall configurado (portas 22, 80, 443, 6443)

### **2. Certificados SSL**
```bash
# Let's Encrypt via certbot
sudo certbot certonly --standalone -d devquote.com.br -d www.devquote.com.br
```

### **3. Imagens Docker**
- `wesleyeduardodev/devquote-backend:latest`
- `wesleyeduardodev/devquote-frontend:latest`

---

## 🚀 Deploy do Zero

### **Passo 1: Clonar Repositório**

```bash
git clone https://github.com/wesleyeduardodev/devquote-infra.git
cd devquote-infra
```

### **Passo 2: Configurar Secrets**

```bash
# Copiar template
cp k8s/secrets.yaml.example k8s/secrets.yaml

# Editar com valores reais (use seu editor preferido)
nano k8s/secrets.yaml
```

**Valores a preencher:**
- Database: User, password
- JWT: Secret key
- Email: SMTP credentials
- AWS: Access keys

### **Passo 3: Criar Namespace**

```bash
kubectl apply -f k8s/namespace.yaml
```

**Saída esperada:**
```
namespace/devquote created
```

### **Passo 4: Criar Secrets**

```bash
# Application secrets
kubectl apply -f k8s/secrets.yaml

# TLS certificate
kubectl create secret tls devquote-tls \
  --cert=/etc/letsencrypt/live/devquote.com.br/fullchain.pem \
  --key=/etc/letsencrypt/live/devquote.com.br/privkey.pem \
  -n devquote
```

**Saída esperada:**
```
secret/devquote-secrets created
secret/devquote-tls created
```

### **Passo 5: Deploy Database**

```bash
kubectl apply -f k8s/database/
```

**Aguardar pod ficar pronto:**
```bash
kubectl wait --for=condition=ready pod -l app=postgres -n devquote --timeout=300s
```

**Verificar:**
```bash
kubectl get pods -n devquote
```

**Saída esperada:**
```
NAME          READY   STATUS    RESTARTS   AGE
postgres-0    1/1     Running   0          60s
```

### **Passo 6: Deploy Backend**

```bash
kubectl apply -f k8s/backend/
```

**Aguardar pods ficarem prontos:**
```bash
kubectl wait --for=condition=ready pod -l app=backend -n devquote --timeout=300s
```

**Verificar logs:**
```bash
kubectl logs -f deployment/backend -n devquote
```

**Procure por:**
```
Started DevquoteApplication in X seconds
```

### **Passo 7: Deploy Frontend**

```bash
kubectl apply -f k8s/frontend/
```

**Aguardar pod ficar pronto:**
```bash
kubectl wait --for=condition=ready pod -l app=frontend -n devquote --timeout=120s
```

### **Passo 8: Deploy Ingress**

```bash
kubectl apply -f k8s/ingress/
```

**Verificar:**
```bash
kubectl get ingress -n devquote
```

**Saída esperada:**
```
NAME               CLASS     HOSTS                                 ADDRESS         PORTS     AGE
devquote-ingress   traefik   devquote.com.br,www.devquote.com.br   31.97.164.223   80, 443   10s
```

### **Passo 9: Deploy Monitoring (Opcional)**

```bash
# Prometheus
kubectl apply -f k8s/monitoring/prometheus/

# Grafana
kubectl apply -f k8s/monitoring/grafana/
```

### **Passo 10: Verificar Tudo**

```bash
# Ver todos os pods
kubectl get pods -n devquote

# Ver todos os serviços
kubectl get svc -n devquote

# Ver ingress
kubectl get ingress -n devquote

# Ver uso de recursos
kubectl top pods -n devquote
```

### **Passo 11: Testar Aplicação**

```bash
# Frontend
curl -I https://devquote.com.br

# Backend health
curl -I https://devquote.com.br/actuator/health

# Acessar no browser
open https://devquote.com.br
```

---

## 🔄 Atualizar Aplicação

### **Opção 1: Atualizar Tag da Imagem**

```bash
# 1. Editar manifesto
nano k8s/backend/deployment.yaml

# Trocar:
# image: wesleyeduardodev/devquote-backend:latest
# Por:
# image: wesleyeduardodev/devquote-backend:v1.2.3

# 2. Aplicar
kubectl apply -f k8s/backend/deployment.yaml

# 3. Acompanhar rollout
kubectl rollout status deployment/backend -n devquote
```

### **Opção 2: Force Pull Latest**

```bash
kubectl rollout restart deployment/backend -n devquote
```

---

## 🐛 Troubleshooting

### **Pod não inicia**

```bash
# Ver eventos
kubectl describe pod <pod-name> -n devquote

# Ver logs
kubectl logs <pod-name> -n devquote

# Casos comuns:
# - ImagePullBackOff: Imagem não existe no Docker Hub
# - CrashLoopBackOff: Aplicação está crashando, ver logs
# - Pending: Falta recursos (RAM/CPU)
```

### **Backend não conecta no banco**

```bash
# Verificar se PostgreSQL está rodando
kubectl get pods -l app=postgres -n devquote

# Ver logs do PostgreSQL
kubectl logs postgres-0 -n devquote

# Testar conexão
kubectl exec -it postgres-0 -n devquote -- psql -U devquote_user -d devquote -c '\dt'

# Verificar secrets
kubectl get secret devquote-secrets -n devquote -o yaml
```

### **Ingress não funciona**

```bash
# Verificar Traefik
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik

# Ver logs do Traefik
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik

# Verificar se porta está aberta
curl -I http://31.97.164.223

# Verificar firewall
sudo ufw status
```

### **Pod está usando muita memória**

```bash
# Ver uso atual
kubectl top pods -n devquote

# Aumentar limite (editar deployment.yaml)
resources:
  limits:
    memory: "1Gi"  # Aumentar conforme necessário

# Aplicar
kubectl apply -f k8s/backend/deployment.yaml
```

### **Aplicação está lenta**

```bash
# Ver métricas
kubectl top pods -n devquote
kubectl top nodes

# Ver logs do backend
kubectl logs -f deployment/backend -n devquote

# Escalar horizontalmente
kubectl scale deployment backend --replicas=3 -n devquote
```

---

## 🔄 Backup e Restore

### **Backup do Banco**

```bash
# Backup manual
kubectl exec -it postgres-0 -n devquote -- \
  pg_dumpall -U devquote_user > backup-$(date +%Y%m%d).sql

# Backup via cron (já configurado no VPS)
# Roda diariamente às 2 AM e envia para S3
```

### **Restore do Banco**

```bash
# Copiar backup para pod
kubectl cp backup-20251027.sql devquote/postgres-0:/tmp/

# Restaurar
kubectl exec -it postgres-0 -n devquote -- \
  psql -U devquote_user -d devquote -f /tmp/backup-20251027.sql
```

---

## 📊 Monitoramento

### **Ver Logs em Tempo Real**

```bash
# Backend (todas as réplicas)
kubectl logs -f deployment/backend -n devquote --all-containers=true

# Backend (pod específico)
kubectl logs -f backend-xxxxx-yyyyy -n devquote

# Frontend
kubectl logs -f deployment/frontend -n devquote

# PostgreSQL
kubectl logs -f postgres-0 -n devquote
```

### **Acessar Métricas**

```bash
# Prometheus (port-forward para acessar localmente)
kubectl port-forward svc/prometheus-service 9090:9090 -n devquote

# Acessar: http://localhost:9090

# Grafana (já exposto via ingress)
# Acessar: https://devquote.com.br/grafana
# User: admin
# Pass: admin123
```

---

## 🔐 Segurança

### **Rotacionar Secrets**

```bash
# 1. Editar secrets
nano k8s/secrets.yaml

# 2. Aplicar novo secret
kubectl apply -f k8s/secrets.yaml

# 3. Restart pods para pegar novos valores
kubectl rollout restart deployment/backend -n devquote
kubectl rollout restart deployment/frontend -n devquote
```

### **Renovar Certificado SSL**

```bash
# Certbot renova automaticamente, mas para forçar:
sudo certbot renew --force-renewal

# Atualizar secret no K8s
kubectl delete secret devquote-tls -n devquote

kubectl create secret tls devquote-tls \
  --cert=/etc/letsencrypt/live/devquote.com.br/fullchain.pem \
  --key=/etc/letsencrypt/live/devquote.com.br/privkey.pem \
  -n devquote

# Restart ingress controller
kubectl rollout restart deployment traefik -n kube-system
```

---

## 🗑️ Remover Aplicação

```bash
# Remover tudo (CUIDADO!)
kubectl delete namespace devquote

# Ou remover componente por componente
kubectl delete -f k8s/backend/
kubectl delete -f k8s/frontend/
kubectl delete -f k8s/database/
kubectl delete -f k8s/ingress/
kubectl delete -f k8s/monitoring/prometheus/
kubectl delete -f k8s/monitoring/grafana/
kubectl delete -f k8s/secrets.yaml
kubectl delete -f k8s/namespace.yaml
```

---

## 📝 Checklist de Deploy

- [ ] K3s instalado e funcionando
- [ ] kubectl configurado
- [ ] Imagens Docker buildadas e no Docker Hub
- [ ] Certificados SSL configurados
- [ ] Secrets.yaml preenchido com valores reais
- [ ] Namespace criado
- [ ] Secrets aplicados (app + TLS)
- [ ] Database deployado e pronto
- [ ] Backend deployado e healthy
- [ ] Frontend deployado e healthy
- [ ] Ingress configurado
- [ ] DNS apontando para VPS
- [ ] Aplicação acessível via HTTPS
- [ ] Monitoring deployado (opcional)
- [ ] Backup configurado

---

## 🆘 Suporte

Se encontrar problemas:

1. Verifique os logs dos pods
2. Verifique eventos do Kubernetes
3. Teste conexões de rede
4. Consulte a documentação de troubleshooting
5. Abra uma issue no GitHub

---

## 📚 Recursos Adicionais

- [Documentação K3s](https://docs.k3s.io/)
- [Kubernetes Docs](https://kubernetes.io/docs/)
- [Traefik Docs](https://doc.traefik.io/traefik/)
- [Argo CD Docs](https://argo-cd.readthedocs.io/)
