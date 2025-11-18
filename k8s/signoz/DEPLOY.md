# 🚀 Deploy SigNoz - 100% GitOps

Deploy do SigNoz via **GitOps puro** (Argo CD). Zero comandos manuais na VPS!

---

## ✅ Pré-requisitos

- [ ] Repositório `devquote-infra` clonado localmente
- [ ] Argo CD funcionando no cluster
- [ ] Git configurado no seu computador

---

## 🎯 **Como Funciona (GitOps)**

```yaml
Fluxo Automatizado:
├─ 1. git commit + push (seu computador)
├─ 2. Argo CD detecta mudanças no GitHub
├─ 3. Argo CD lê k8s/argocd/signoz-application.yaml
├─ 4. Argo CD cria Application do SigNoz
├─ 5. Application instala SigNoz via Helm
├─ 6. Argo CD aplica k8s/signoz/middleware.yaml e ingress.yaml
└─ 7. ✅ SigNoz funcionando!

Tempo total: ~10-15 minutos
Comandos manuais: ZERO! 🎉
```

---

## 📋 Passo a Passo

### **Etapa 1: Commit e Push**

```bash
# No seu computador
cd devquote-infra

# Verificar arquivos criados
git status

# Deve aparecer:
# - k8s/argocd/signoz-application.yaml (nova)
# - k8s/signoz/ (novo diretório)
# - k8s/backend/deployment.yaml (modificado)

# Adicionar tudo
git add .

# Commit
git commit -m "feat: adiciona SigNoz para observabilidade completa

- Adiciona Argo CD Application do SigNoz via Helm
- Configura OpenTelemetry no backend
- Cria Ingress em /signoz com TLS
- Middleware Traefik para strip prefix
- Otimizado para VPS 8GB RAM (retenção 7d)
- Grafana permanece ativo para comparação
- 100% GitOps (zero comandos manuais)"

# Push
git push origin main
```

**⏱️ Tempo:** 2 minutos

---

### **Etapa 2: Aguardar Argo CD Sincronizar**

```bash
# Acompanhar sincronização via UI
https://devquote.com.br/argocd

# Status deve mudar:
OutOfSync → Syncing → Synced

# Ou via CLI (opcional):
ssh vps "watch kubectl get application -n argocd"
```

**⏱️ Tempo:** 3-5 minutos

**O que acontece:**
1. Argo CD detecta `k8s/argocd/signoz-application.yaml`
2. Cria Application "signoz" no namespace argocd
3. Application instala SigNoz via Helm chart
4. Argo CD aplica `k8s/signoz/middleware.yaml` e `ingress.yaml`

---

### **Etapa 3: Verificar Pods (Opcional)**

```bash
# SSH na VPS (apenas para acompanhar)
ssh vps

# Ver pods do SigNoz
kubectl get pods -n devquote -l app.kubernetes.io/instance=signoz

# Aguardar todos ficarem Running (5-10 min):
# NAME                                    READY   STATUS
# signoz-clickhouse-0                     1/1     Running
# signoz-query-service-xxx                1/1     Running
# signoz-otel-collector-xxx               1/1     Running
# signoz-frontend-xxx                     1/1     Running
# signoz-alertmanager-xxx                 1/1     Running

# Backend deve reiniciar automaticamente (Argo CD)
kubectl get pods -n devquote -l app=backend
```

**⏱️ Tempo:** 10 minutos

---

### **Etapa 4: Acessar SigNoz**

```
URL: https://devquote.com.br/signoz

Primeiro acesso:
1. Criar conta admin
2. Email: wesleyeduardo.dev@gmail.com
3. Senha: <escolher senha forte>
4. Confirmar
```

**⏱️ Tempo:** 2 minutos

---

### **Etapa 5: Validar Coleta de Dados**

```bash
# Fazer requisições no backend
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
[ ] git commit + push
[ ] Argo CD sincronizou automaticamente
[ ] Application "signoz" criada
[ ] Todos os pods do SigNoz em Running
[ ] Backend reiniciou com OTEL
[ ] Ingress acessível em /signoz
[ ] Conta admin criada no SigNoz
[ ] Service "devquote-backend" aparecendo
[ ] Traces sendo coletados
[ ] Logs sendo coletados
[ ] Correlação logs ↔ traces funcionando
```

---

## 🎯 Tempo Total

**~20-25 minutos** (a maior parte é aguardando pods iniciarem)

**Comandos manuais:** ZERO! ✅

---

## 🔄 Nova VPS? Zero Comandos Manuais!

```bash
# Setup de nova VPS:
1. Instalar K3s
2. Instalar Argo CD
3. Aplicar argocd/application.yaml (bootstrap inicial)
4. git push → tudo automático! ✅

# SigNoz será instalado automaticamente via GitOps!
```

---

## 🔍 Monitoramento Pós-Deploy

### Ver Recursos Consumidos

```bash
ssh vps

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

# Ver Applications do Argo CD
kubectl get application -n argocd

# Eventos recentes
kubectl get events -n devquote --sort-by='.lastTimestamp' | head -20
```

---

## ⚠️ Troubleshooting

### Argo CD não sincronizou

**Causa:** Timeout ou erro de parse

**Solução:**
```bash
# Ver logs do Argo CD
ssh vps "kubectl logs -n argocd deployment/argocd-application-controller"

# Forçar sync manual (via UI)
https://devquote.com.br/argocd
# Clicar em "signoz" → SYNC
```

### Pods não iniciam (Pending/CrashLoopBackOff)

**Causa:** Falta de recursos (RAM/CPU)

**Solução:**
```bash
# Ver motivo
kubectl describe pod <nome-do-pod> -n devquote

# Se for OOMKilled, desabilitar Grafana temporariamente:
kubectl scale deployment grafana -n devquote --replicas=0
kubectl scale deployment prometheus -n devquote --replicas=0
kubectl scale statefulset loki -n devquote --replicas=0
```

### Backend não envia dados

**Causa:** OTEL Collector não está acessível

**Solução:**
```bash
# Verificar logs do backend
kubectl logs -f deployment/backend -n devquote | grep -i otel

# Deve aparecer:
# "OpenTelemetry Javaagent started"

# Verificar conectividade
kubectl exec -it deployment/backend -n devquote -- \
  curl -v http://signoz-otel-collector.devquote.svc.cluster.local:4318
```

---

## 🔄 Rollback

### Desinstalar SigNoz

```bash
# No seu computador
cd devquote-infra

# Deletar Application
rm k8s/argocd/signoz-application.yaml

# Deletar manifestos complementares
rm -rf k8s/signoz/

# Reverter backend
git revert <commit-hash>

# Commit + push
git add .
git commit -m "revert: remove SigNoz"
git push

# Argo CD remove automaticamente! ✅
```

---

## 🎉 Sucesso!

SigNoz rodando em paralelo com Grafana via **100% GitOps**!

**URLs:**
- **Grafana:** https://devquote.com.br/grafana (continua ativo)
- **SigNoz:** https://devquote.com.br/signoz (novo!)

**Próximos passos:**
1. Testar por 1-2 semanas
2. Comparar experiência (Grafana vs SigNoz)
3. Criar dashboards personalizados
4. Configurar alertas
5. Decidir qual manter

---

**Última atualização:** 2025-01-18
**Versão:** 2.0.0 (100% GitOps - Zero Comandos Manuais)
