# Grafana Tempo - Distributed Tracing

Grafana Tempo é um sistema de distributed tracing de alto desempenho e baixo custo que se integra perfeitamente com a stack Grafana/Prometheus/Loki.

## O que é Distributed Tracing?

Distributed tracing permite visualizar o fluxo completo de uma requisição através de todos os componentes do sistema, mostrando:
- Tempo gasto em cada método/operação
- Gargalos de performance
- Erros e exceções
- Chamadas a bancos de dados e APIs externas

## Arquitetura

```
Backend (Spring Boot)
    ↓
OpenTelemetry Agent (exporta traces via OTLP)
    ↓
Tempo (armazena traces)
    ↓
Grafana (visualiza traces + correlação com logs)
```

## Componentes Instalados

### Deployment (`deployment.yaml`)
- **Imagem:** `grafana/tempo:2.8.0`
- **Portas:**
  - `3200`: HTTP API
  - `4318`: OTLP HTTP (recebe traces do backend)
  - `4317`: OTLP gRPC
- **Recursos:**
  - CPU: 100m request, 200m limit
  - RAM: 128Mi request, 256Mi limit

### Storage (`pvc.yaml`)
- **Volume:** 10Gi
- **Backend:** Local storage
- **Retenção:** 72 horas (configurável em `configmap.yaml`)

### Configuração (`configmap.yaml`)
- **Receivers:** OTLP HTTP/gRPC
- **Metrics Generator:** Envia métricas para Prometheus
- **Compactor:** Compacta traces antigos
- **Query:** Interface de busca

### Grafana Datasource (`grafana-datasource.yaml`)
- **Correlação Logs ↔ Traces:** Conecta automaticamente traces do Tempo com logs do Loki
- **Service Map:** Mapa de serviços usando Prometheus
- **Node Graph:** Visualização de dependências

## Como Usar

### 1. Verificar Status no Argo CD
```bash
# Via UI: https://devquote.com.br/argocd
# Verificar se todos os recursos estão "Synced" e "Healthy"
```

### 2. Acessar Grafana
```
URL: https://devquote.com.br/grafana
User: admin
Pass: admin123
```

### 3. Explorar Traces
1. Menu: **Explore**
2. Selecionar datasource: **Tempo**
3. Fazer requisições na API do DevQuote
4. Buscar traces:
   - Por TraceID
   - Por Service Name (`devquote-backend`)
   - Por Duração (ex: `> 1s`)
   - Por Status (`error`, `ok`)

### 4. Correlacionar Logs com Traces
Ao visualizar um trace no Grafana:
- Clique em qualquer span
- Clique em **"Logs for this span"**
- Grafana automaticamente busca os logs relacionados no Loki

## Endpoints

### Backend → Tempo
```
Backend envia traces para:
http://tempo.devquote.svc.cluster.local:4318/v1/traces
```

### Grafana → Tempo
```
Grafana consulta traces em:
http://tempo.devquote.svc.cluster.local:3200
```

## Configuração no Backend

### Environment Variables
```yaml
OTEL_EXPORTER_OTLP_ENDPOINT: http://tempo.devquote.svc.cluster.local:4318
OTEL_SERVICE_NAME: devquote-backend
OTEL_METRICS_EXPORTER: none
```

### Sampling
Por padrão, **100% das requisições** são traced (desenvolvimento).

Para produção, ajustar no `application.yml`:
```yaml
management:
  tracing:
    sampling:
      probability: 0.1  # 10% das requisições
```

## Logs com TraceID

Todos os logs do backend incluem `traceId` e `spanId`:
```
[trace:64b2f7c8a3e1d2f9 span:a1b2c3d4e5f6]
```

Isso permite buscar logs de um trace específico!

## Troubleshooting

### Traces não aparecem no Grafana
1. Verificar se Tempo está Running:
   ```bash
   kubectl get pods -n devquote | grep tempo
   ```

2. Verificar logs do Tempo:
   ```bash
   kubectl logs -f deployment/tempo -n devquote
   ```

3. Verificar se backend está enviando traces:
   ```bash
   kubectl logs -f deployment/backend -n devquote | grep -i "otel\|trace"
   ```

### Tempo com erro
1. Verificar se PVC foi criado:
   ```bash
   kubectl get pvc tempo-pvc -n devquote
   ```

2. Verificar ConfigMap:
   ```bash
   kubectl get configmap tempo-config -n devquote
   ```

### Grafana não reconhece Tempo
1. Verificar se datasource foi criado:
   ```bash
   kubectl get configmap grafana-tempo-datasource -n devquote
   ```

2. Reiniciar Grafana:
   ```bash
   kubectl rollout restart deployment/grafana -n devquote
   ```

## Métricas

Tempo gera automaticamente métricas de spans e envia para Prometheus:
- Duração de operações
- Taxa de erro
- Throughput

Essas métricas aparecem no Prometheus com prefixo `tempo_`.

## Retenção de Dados

Configuração atual:
- **Traces:** 72 horas (3 dias)
- **Compacted Blocks:** 1 hora

Para aumentar retenção, editar `k8s/tempo/configmap.yaml`:
```yaml
compactor:
  compaction:
    block_retention: 168h  # 7 dias
```

## Recursos Consumidos

- **CPU:** ~10-15% (100-150m)
- **RAM:** ~150-200Mi
- **Storage:** ~5-10Gi (dependendo do volume de traces)

## Documentação Oficial

- [Grafana Tempo Docs](https://grafana.com/docs/tempo/latest/)
- [OpenTelemetry Java](https://opentelemetry.io/docs/languages/java/)
- [Micrometer Tracing](https://micrometer.io/docs/tracing)

## Próximos Passos

1. ✅ Tempo instalado e funcionando
2. ✅ Backend enviando traces
3. ✅ Grafana visualizando traces
4. ⬜ Adicionar tracing customizado no código
5. ⬜ Criar dashboards específicos de APM
6. ⬜ Configurar alertas baseados em traces
