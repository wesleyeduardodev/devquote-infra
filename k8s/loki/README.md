# Loki - Sistema de Logs Centralizado

Stack PLG (Promtail + Loki + Grafana) para centralização e persistência de logs.

---

## Arquitetura

```
Apps (Backend/Frontend)
    ↓ logs stdout/stderr
Kubernetes captura
    ↓ /var/log/pods/
Promtail (DaemonSet)
    ↓ coleta + labels
Loki (StatefulSet)
    ↓ armazena (PersistentVolume)
Grafana
    ↓ visualiza
Você!
```

---

## Componentes

### 1. Loki (StatefulSet)
- **Imagem:** grafana/loki:3.0.0
- **Recursos:** 256-512Mi RAM, 100-500m CPU
- **Armazenamento:** 5Gi PersistentVolume
- **Retenção:** 15 dias (360h)
- **Porta:** 3100 (HTTP), 9096 (gRPC)

### 2. Promtail (DaemonSet)
- **Imagem:** grafana/promtail:3.0.0
- **Recursos:** 64-128Mi RAM, 50-200m CPU
- **Função:** Coleta logs de todos os pods
- **Labels automáticos:** namespace, app, pod, container, node

### 3. Grafana Datasource
- **URL:** http://loki:3100
- **Tipo:** Loki
- **Features:** LogQL, labels, correlação com métricas

---

## Deploy

```bash
# Aplicar todos os manifestos
kubectl apply -f k8s/loki/

# Verificar status
kubectl get pods -n devquote | grep -E "loki|promtail"

# Deve mostrar:
# loki-0                     1/1     Running
# promtail-xxxxx             1/1     Running
```

---

## Uso no Grafana

### 1. Acessar Grafana
https://devquote.com.br/grafana

### 2. Explore → Loki

### 3. Queries LogQL

**Todos os logs do backend:**
```logql
{namespace="devquote", app="backend"}
```

**Apenas erros:**
```logql
{app="backend"} |= "ERROR"
```

**Logs de uma classe específica:**
```logql
{app="backend", class="TaskServiceImpl"}
```

**Logs com filtro de conteúdo:**
```logql
{app="backend"} |= "taskId" |= "123"
```

**Taxa de erros por minuto:**
```logql
rate({app="backend"} |= "ERROR" [1m])
```

**Últimas 100 linhas:**
```logql
{app="backend"} | limit 100
```

---

## Verificar Logs

### Ver logs do Loki
```bash
kubectl logs -f loki-0 -n devquote
```

### Ver logs do Promtail
```bash
kubectl logs -f daemonset/promtail -n devquote
```

### Verificar se Promtail está enviando logs
```bash
# Acessar metrics do Promtail
kubectl port-forward daemonset/promtail 9080:9080 -n devquote

# Abrir no navegador: http://localhost:9080/metrics
# Procurar por: promtail_sent_entries_total
```

### Verificar se Loki está recebendo logs
```bash
# Query direta via API
kubectl exec -it loki-0 -n devquote -- wget -qO- 'http://localhost:3100/loki/api/v1/query?query={namespace="devquote"}' | jq
```

---

## Troubleshooting

### Loki não está iniciando

```bash
# Ver logs
kubectl logs -f loki-0 -n devquote

# Verificar PVC
kubectl get pvc loki-data -n devquote

# Verificar se tem espaço em disco
kubectl exec -it loki-0 -n devquote -- df -h /loki
```

### Promtail não está coletando logs

```bash
# Verificar RBAC
kubectl get clusterrolebinding promtail

# Verificar se tem acesso aos logs
kubectl exec -it $(kubectl get pods -n devquote -l app=promtail -o name | head -1) -n devquote -- ls -la /var/log/pods/devquote*

# Ver configuração
kubectl get cm promtail-config -n devquote -o yaml
```

### Grafana não mostra datasource Loki

```bash
# Verificar se ConfigMap foi aplicado
kubectl get cm grafana-loki-datasource -n devquote

# Reiniciar Grafana para recarregar datasources
kubectl rollout restart deployment/grafana -n devquote

# Verificar logs do Grafana
kubectl logs -f deployment/grafana -n devquote
```

### Logs não aparecem no Grafana

```bash
# 1. Verificar se Promtail está enviando
kubectl logs -f daemonset/promtail -n devquote | grep -i "sent"

# 2. Verificar se Loki está recebendo
kubectl logs -f loki-0 -n devquote | grep -i "ingester"

# 3. Testar query direta no Loki
kubectl port-forward svc/loki 3100:3100 -n devquote
curl 'http://localhost:3100/loki/api/v1/query?query={namespace="devquote"}' | jq

# 4. Verificar labels disponíveis
curl 'http://localhost:3100/loki/api/v1/labels' | jq
```

---

## Configurações Avançadas

### Aumentar retenção para 30 dias

Editar `loki-config.yaml`:
```yaml
limits_config:
  retention_period: 720h  # 30 dias

chunk_store_config:
  max_look_back_period: 720h

table_manager:
  retention_period: 720h
```

Aplicar:
```bash
kubectl apply -f k8s/loki/loki-config.yaml
kubectl rollout restart statefulset/loki -n devquote
```

### Aumentar armazenamento

Editar `loki-pvc.yaml`:
```yaml
resources:
  requests:
    storage: 10Gi  # Era 5Gi
```

**ATENÇÃO:** Não é possível reduzir PVC depois de criado!

### Ajustar recursos do Loki

Editar `loki-statefulset.yaml`:
```yaml
resources:
  requests:
    memory: "512Mi"  # Era 256Mi
    cpu: "200m"      # Era 100m
  limits:
    memory: "1Gi"    # Era 512Mi
    cpu: "1000m"     # Era 500m
```

---

## Queries Úteis

### Logs de erro agrupados por classe
```logql
sum by (class) (count_over_time({app="backend"} |= "ERROR" [1h]))
```

### Top 10 classes com mais logs
```logql
topk(10, sum by (class) (count_over_time({app="backend"} [1h])))
```

### Logs de autenticação
```logql
{app="backend"} |~ "login|logout|authentication"
```

### Logs de requisições HTTP
```logql
{app="backend"} |~ "GET|POST|PUT|DELETE"
```

### Logs slow queries (PostgreSQL)
```logql
{app="backend"} |~ "duration=\\d{4,}" # 1000ms+
```

---

## Dashboards Recomendados

Importar do Grafana:
- **13639** - Loki Dashboard
- **12019** - Loki Logs Panel
- **15141** - Kubernetes Logs (Loki)

---

## Backup e Restore

### Backup do índice e chunks
```bash
# Criar backup do PVC
kubectl exec -it loki-0 -n devquote -- tar czf /tmp/loki-backup.tar.gz /loki

# Copiar para local
kubectl cp devquote/loki-0:/tmp/loki-backup.tar.gz ./loki-backup.tar.gz
```

### Restore
```bash
# Copiar backup para pod
kubectl cp ./loki-backup.tar.gz devquote/loki-0:/tmp/

# Restaurar
kubectl exec -it loki-0 -n devquote -- tar xzf /tmp/loki-backup.tar.gz -C /

# Reiniciar Loki
kubectl rollout restart statefulset/loki -n devquote
```

---

## Recursos Consumidos

| Componente | RAM (request) | RAM (limit) | CPU (request) | CPU (limit) |
|------------|---------------|-------------|---------------|-------------|
| Loki | 256Mi | 512Mi | 100m | 500m |
| Promtail | 64Mi | 128Mi | 50m | 200m |
| **TOTAL** | **320Mi** | **640Mi** | **150m** | **700m** |

**Disco:** 5Gi (PersistentVolume)

---

## Referências

- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [Promtail Configuration](https://grafana.com/docs/loki/latest/clients/promtail/configuration/)
- [LogQL Cheat Sheet](https://megamorf.gitlab.io/cheat-sheets/loki/)
