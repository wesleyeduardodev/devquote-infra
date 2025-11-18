# SigNoz - Observabilidade Completa

Sistema de observabilidade all-in-one para monitoramento de traces, logs e métricas do DevQuote.

---

## 🎯 O que é SigNoz?

SigNoz é uma plataforma open-source de **APM (Application Performance Monitoring)** que unifica:
- **Traces** (rastreamento de requisições)
- **Logs** (logs da aplicação)
- **Métricas** (performance, latência, error rate)

Tudo em **uma única interface** com correlação automática.

---

## 📊 Recursos Configurados

### Otimizado para VPS 8GB RAM

| Componente | RAM | CPU | Função |
|------------|-----|-----|--------|
| ClickHouse | 400-700Mi | 300-500m | Banco de dados OLAP |
| Query Service | 150-250Mi | 100-200m | API de queries |
| OTEL Collector | 150-250Mi | 100-200m | Coleta de telemetria |
| Frontend | 100-150Mi | 50-100m | Interface web |
| AlertManager | 50-100Mi | 50-100m | Gerenciamento de alertas |
| **TOTAL** | **~1-1.6GB** | **~600-1000m** | |

### Retenção de Dados
- **Traces:** 7 dias
- **Logs:** 7 dias
- **Métricas:** 7 dias

**Armazenamento:** ~10GB (PVC ClickHouse)

---

## 🚀 Deploy (GitOps)

### 1. Aplicar Argo CD Application

O SigNoz será instalado automaticamente via Argo CD quando você aplicar a Application.

```bash
# SSH na VPS
ssh root@31.97.164.223

# Aplicar Application do SigNoz no Argo CD
kubectl apply -f /root/devquote-infra/argocd/signoz-application.yaml

# Aguardar Argo CD sincronizar (3-5 min)
watch kubectl get pods -n devquote -l app.kubernetes.io/instance=signoz
```

### 2. Verificar Deploy

```bash
# Ver status dos pods
kubectl get pods -n devquote -l app.kubernetes.io/instance=signoz

# Deve aparecer (aguardar todos ficarem Running):
# signoz-clickhouse-0            1/1     Running
# signoz-query-service-xxx       1/1     Running
# signoz-otel-collector-xxx      1/1     Running
# signoz-frontend-xxx            1/1     Running
# signoz-alertmanager-xxx        1/1     Running
```

### 3. Aplicar Ingress e Middleware

```bash
# Aplicar middleware do Traefik
kubectl apply -f /root/devquote-infra/k8s/signoz/middleware.yaml

# Aplicar Ingress
kubectl apply -f /root/devquote-infra/k8s/signoz/ingress.yaml

# Verificar Ingress
kubectl get ingress signoz-ingress -n devquote
```

### 4. Acessar SigNoz

```
URL: https://devquote.com.br/signoz

Primeiro acesso:
1. Criar conta admin
2. Email: seu@email.com
3. Senha: <escolher senha forte>
```

---

## 🔧 Backend - OpenTelemetry

O backend foi atualizado para enviar dados para o SigNoz via **OpenTelemetry**.

### Como Funciona

```yaml
Backend Deployment:
├─ InitContainer: baixa opentelemetry-javaagent.jar
├─ Container principal:
│  ├─ JAVA_TOOL_OPTIONS: -javaagent:/otel/opentelemetry-javaagent.jar
│  ├─ OTEL_SERVICE_NAME: devquote-backend
│  ├─ OTEL_EXPORTER_OTLP_ENDPOINT: http://signoz-otel-collector:4318
│  └─ Envia: traces, logs, métricas → SigNoz
```

### Aplicar via GitOps

O backend será atualizado automaticamente pelo Argo CD após commit:

```bash
# No seu computador local (repositório devquote-infra)
git add .
git commit -m "feat: adiciona SigNoz e OpenTelemetry no backend"
git push

# Argo CD aplica automaticamente em ~3 minutos
```

---

## 📋 Validação Pós-Deploy

### 1. Verificar Coleta de Dados

```bash
# Fazer requisições no backend
curl https://devquote.com.br/api/auth/login

# Aguardar 1-2 minutos
```

### 2. No SigNoz UI

1. Acesse: https://devquote.com.br/signoz
2. Vá em **"Services"**
   - Deve aparecer: `devquote-backend`
3. Vá em **"Traces"**
   - Deve aparecer traces das requisições
4. Vá em **"Logs"**
   - Deve aparecer logs da aplicação

### 3. Criar Alertas Básicos

No SigNoz, criar alertas:
- **Latência P95 > 2s**
- **Error rate > 5%**
- **Request rate anormal**

---

## 🔍 Monitoramento

### Ver Consumo de Recursos

```bash
# Recursos do SigNoz
kubectl top pods -n devquote -l app.kubernetes.io/instance=signoz

# Recursos totais do namespace
kubectl top pods -n devquote
```

### Ver Logs

```bash
# ClickHouse (mais pesado)
kubectl logs -f statefulset/signoz-clickhouse -n devquote

# Query Service
kubectl logs -f deployment/signoz-query-service -n devquote

# OTEL Collector (recebe dados do backend)
kubectl logs -f deployment/signoz-otel-collector -n devquote
```

### Ver Eventos

```bash
kubectl get events -n devquote --sort-by='.lastTimestamp' | grep signoz
```

---

## ⚠️ Troubleshooting

### Pods não iniciam (Pending)

**Causa:** Falta de recursos (RAM/CPU)

**Solução:**
```bash
# Ver recursos disponíveis
kubectl top nodes

# Se RAM > 85%, considerar desabilitar Grafana:
kubectl scale deployment grafana -n devquote --replicas=0
kubectl scale deployment prometheus -n devquote --replicas=0
kubectl scale statefulset loki -n devquote --replicas=0
```

### Backend não envia dados

**Causa:** OTEL Collector não está acessível

**Verificar:**
```bash
# Service do OTEL Collector
kubectl get svc -n devquote | grep otel-collector

# Deve aparecer:
# signoz-otel-collector   ClusterIP   10.x.x.x   <none>   4318/TCP

# Testar conectividade do backend
kubectl exec -it deployment/backend -n devquote -- curl http://signoz-otel-collector:4318
```

### Ingress não funciona (404)

**Causa:** Nome do service do frontend incorreto

**Verificar:**
```bash
# Listar services do SigNoz
kubectl get svc -n devquote -l app.kubernetes.io/instance=signoz

# Identificar nome correto do frontend
# Atualizar ingress.yaml se necessário
```

---

## 🔄 Rollback

### Desinstalar SigNoz

```bash
# Deletar Application do Argo CD
kubectl delete application signoz -n argocd

# Argo CD remove automaticamente todos os recursos

# Remover Ingress e Middleware
kubectl delete -f k8s/signoz/ingress.yaml
kubectl delete -f k8s/signoz/middleware.yaml

# Remover OTEL do backend
# Reverter commit ou aplicar deployment antigo
```

### Voltar Backend para Versão Anterior

```bash
# No repositório local
git revert HEAD~1  # reverte último commit
git push

# Argo CD aplica automaticamente
```

---

## 📊 Comparativo: Grafana vs SigNoz

Após 1-2 semanas de teste, avaliar:

| Critério | Grafana | SigNoz |
|----------|---------|--------|
| Facilidade de uso | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Dashboards prontos | ❌ | ✅ |
| Correlação logs/traces | Manual | Automática |
| Consumo de recursos | Menor | Maior |
| Queries | 3 linguagens | SQL-like |

**Decisão:**
- Se SigNoz for melhor: desabilitar Grafana (scale to 0)
- Se Grafana for melhor: desinstalar SigNoz

---

## 🎯 Links Úteis

- **SigNoz:** https://devquote.com.br/signoz
- **Grafana (atual):** https://devquote.com.br/grafana
- **Documentação SigNoz:** https://signoz.io/docs/
- **OpenTelemetry Java:** https://opentelemetry.io/docs/languages/java/

---

## 📝 Notas

- **Grafana permanece ativo** durante testes
- **Prometheus continua coletando** métricas
- **Backend envia dados para ambos** (Prometheus + SigNoz)
- **Zero downtime** durante migração

---

**Última atualização:** 2025-01-18
**Versão:** 1.0.0 (Deploy Inicial)
