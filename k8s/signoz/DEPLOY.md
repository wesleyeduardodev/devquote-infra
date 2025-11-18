# 🚀 Deploy SigNoz - Guia Rápido

Guia passo a passo para deploy do SigNoz via **GitOps** (Argo CD).

---

## ✅ Pré-requisitos

- [ ] Repositório `devquote-infra` clonado localmente
- [ ] Acesso SSH à VPS (31.97.164.223)
- [ ] Argo CD funcionando no cluster
- [ ] Grafana/Prometheus ativos (serão mantidos)

---

## 📋 Passo a Passo

### **Etapa 1: Commit e Push (Seu Computador)**

```bash
# Navegar para o repositório
cd devquote-infra

# Verificar arquivos criados
git status

# Deve aparecer:
# - argocd/signoz-application.yaml (nova)
# - k8s/signoz/ (novo diretório)
# - k8s/backend/deployment.yaml (modificado)

# Adicionar tudo
git add .

# Commit
git commit -m "feat: adiciona SigNoz para observabilidade completa

- Adiciona Argo CD Application do SigNoz
- Configura OpenTelemetry no backend
- Cria Ingress em /signoz
- Otimizado para VPS 8GB RAM
- Grafana permanece ativo para comparação"

# Push
git push origin main
```

**⏱️ Tempo:** 2 minutos

---

### **Etapa 2: Aplicar Application no Argo CD (VPS)**

```bash
# SSH na VPS
ssh root@31.97.164.223

# Ir para o repositório (se já tiver clonado)
cd /root/devquote-infra

# Ou clonar (se for primeira vez)
git clone https://github.com/wesleyeduardodev/devquote-infra
cd devquote-infra

# Pull das mudanças
git pull

# Aplicar Application do SigNoz
kubectl apply -f argocd/signoz-application.yaml

# Verificar se Application foi criada
kubectl get application signoz -n argocd
```

**⏱️ Tempo:** 2 minutos

---

### **Etapa 3: Aguardar Argo CD Sincronizar**

```bash
# Acompanhar sincronização
watch kubectl get application signoz -n argocd

# Status deve mudar:
# OutOfSync → Syncing → Synced

# Ou ver na UI do Argo CD:
# https://devquote.com.br/argocd
```

**⏱️ Tempo:** 3-5 minutos (download das imagens)

---

### **Etapa 4: Verificar Pods**

```bash
# Ver pods do SigNoz
kubectl get pods -n devquote -l app.kubernetes.io/instance=signoz

# Aguardar todos ficarem Running (pode levar 5-10 min):
# NAME                                    READY   STATUS
# signoz-clickhouse-0                     1/1     Running
# signoz-query-service-xxx                1/1     Running
# signoz-otel-collector-xxx               1/1     Running
# signoz-frontend-xxx                     1/1     Running
# signoz-alertmanager-xxx                 1/1     Running
```

**⏱️ Tempo:** 5-10 minutos

---

### **Etapa 5: Aplicar Ingress e Middleware**

```bash
# Aplicar middleware do Traefik
kubectl apply -f k8s/signoz/middleware.yaml

# Aplicar Ingress
kubectl apply -f k8s/signoz/ingress.yaml

# Verificar Ingress
kubectl get ingress signoz-ingress -n devquote
```

**⏱️ Tempo:** 1 minuto

---

### **Etapa 6: Aguardar Backend Reiniciar (OTEL)**

O Argo CD vai detectar mudanças no `k8s/backend/deployment.yaml` e reiniciar o backend.

```bash
# Acompanhar reinício do backend
watch kubectl get pods -n devquote -l app=backend

# O pod vai:
# 1. Terminating (pod antigo)
# 2. Init:0/1 (baixando OTEL agent)
# 3. Running (pod novo com OTEL)

# Ver logs do initContainer
kubectl logs -f deployment/backend -n devquote -c download-otel-agent

# Ver logs do backend (deve aparecer logs do OTEL)
kubectl logs -f deployment/backend -n devquote
```

**⏱️ Tempo:** 2-3 minutos

---

### **Etapa 7: Acessar SigNoz**

```bash
# Abrir no navegador:
https://devquote.com.br/signoz

# Primeiro acesso:
1. Criar conta admin
2. Email: wesleyeduardo.dev@gmail.com
3. Senha: <escolher senha forte>
4. Confirmar
```

**⏱️ Tempo:** 2 minutos

---

### **Etapa 8: Validar Coleta de Dados**

```bash
# Fazer algumas requisições no backend
curl https://devquote.com.br/api/tasks
curl https://devquote.com.br/api/projects

# Aguardar 1-2 minutos

# No SigNoz UI:
1. Ir em "Services"
   → Deve aparecer: devquote-backend

2. Ir em "Traces"
   → Deve aparecer traces das requisições

3. Ir em "Logs"
   → Deve aparecer logs da aplicação

4. Clicar em um trace
   → Ver flame graph detalhado
   → Clicar em "View Logs" → correlação automática!
```

**⏱️ Tempo:** 5 minutos

---

## ✅ Checklist Final

```
[ ] Application do SigNoz criada no Argo CD
[ ] Todos os pods do SigNoz em Running
[ ] Ingress acessível em /signoz
[ ] Backend reiniciou com OTEL
[ ] Conta admin criada no SigNoz
[ ] Service "devquote-backend" aparecendo
[ ] Traces sendo coletados
[ ] Logs sendo coletados
[ ] Correlação logs ↔ traces funcionando
```

---

## 🎯 Tempo Total

**~15-20 minutos** (a maior parte é aguardando pods iniciarem)

---

## 🔍 Monitoramento Pós-Deploy

### Ver Recursos Consumidos

```bash
# Recursos do SigNoz
kubectl top pods -n devquote -l app.kubernetes.io/instance=signoz

# Recursos totais
kubectl top pods -n devquote

# Uso do node (VPS)
kubectl top node
```

### Verificar Saúde

```bash
# Todos os pods devem estar Running
kubectl get pods -n devquote

# Eventos recentes
kubectl get events -n devquote --sort-by='.lastTimestamp' | head -20
```

---

## ⚠️ Se Algo Der Errado

### Pods não iniciam (Pending/CrashLoopBackOff)

```bash
# Ver motivo
kubectl describe pod <nome-do-pod> -n devquote

# Ver logs
kubectl logs <nome-do-pod> -n devquote

# Se for falta de recursos (OOMKilled):
# Desabilitar Grafana temporariamente:
kubectl scale deployment grafana -n devquote --replicas=0
kubectl scale deployment prometheus -n devquote --replicas=0
kubectl scale statefulset loki -n devquote --replicas=0
```

### Backend não envia dados

```bash
# Verificar logs do backend
kubectl logs -f deployment/backend -n devquote | grep -i otel

# Deve aparecer:
# "OpenTelemetry Javaagent started"
# "Exporting spans to http://signoz-otel-collector:4318"

# Verificar conectividade
kubectl exec -it deployment/backend -n devquote -- \
  curl -v http://signoz-otel-collector.devquote.svc.cluster.local:4318
```

### Rollback Completo

```bash
# Deletar Application do Argo CD
kubectl delete application signoz -n argocd

# Reverter commit
cd /root/devquote-infra
git revert HEAD
git push

# Argo CD aplica automaticamente
```

---

## 🎉 Sucesso!

SigNoz rodando em paralelo com Grafana!

**Próximos passos:**
1. Testar por 1-2 semanas
2. Comparar experiência (Grafana vs SigNoz)
3. Criar dashboards personalizados
4. Configurar alertas
5. Decidir qual manter

**Grafana permanece funcionando!**
- https://devquote.com.br/grafana (continua ativo)
- https://devquote.com.br/signoz (novo!)

---

**Última atualização:** 2025-01-18
