---
date: 2026-08-13
layout: post
title: "Poor Man's Fury - Parte 3"
subtitle: Traces, métricas y error budgets
---

Escenas del episodio anterior: en la Parte 2 quedó una app corriendo en `fury-dev` con sus secrets inyectados por Vault, métricas y logs apareciendo solos en Grafana. El cierre apuntaba a Backstage como próximo paso: un catálogo, una UI, algo que se pareciera a un producto.

En el medio pasaron cosas y, repensando el plan, no tiene sentido construir un catálogo lindo sobre un sistema que no podés ver por dentro. Y "ver por dentro" es exactamente el tema con el que abrió esta serie: la diferencia entre que un LLM te sugiera un diagnóstico y poder confirmarlo vos porque entendés qué está pasando. Prometheus y Loki (Parte 2) te dicen *qué* está roto. Faltaba el tercer pilar: *por qué*, siguiendo un request específico a través de todo el sistema.

Así surgió la fase *4.5*: Traces con Tempo, la propia app instrumentada con OpenTelemetry, métricas custom con un dashboard RED, correlación trace <> logs, y SLOs con error budget de verdad.

## Fase 4.5 - El tercer pilar: traces con Tempo

> **Tracing**: mientras que una métrica te dice "la latencia p99 subió" y un log te dice "esta request específica falló", un trace te muestra el camino completo de una request particular a través de todos los servicios que tocó, con cuánto tardó cada tramo. Es la herramienta para responder "¿por qué esta request en particular tardó 3 segundos?" en un sistema con más de un salto.

```bash
cat > tempo-values.yaml << 'EOF'
tempo:
  storage:
    trace:
      backend: local
persistence:
  enabled: true
  size: 10Gi
EOF

helm install tempo grafana/tempo \
  --namespace fury-monitoring \
  --values tempo-values.yaml
```

El chart tira `WARNING: This chart is deprecated` al instalar — no bloquea nada, queda anotado como deuda técnica (mismo tipo de aviso que ya había con Promtail). El pod pasa por un `0/1 Running` con un `503` de readiness probe mientras termina de levantar, se resuelve solo en menos de un minuto.

> **Issue 01**: el plan original tenía el puerto del datasource de Tempo mal copiado — `3100`, que en realidad es el puerto de Loki. El puerto real de la API HTTP de Tempo es `3200`. Con el puerto equivocado, Grafana no tira un error claro, simplemente "Data source is working" nunca aparece y el datasource queda mudo.

```
Configuration > Data Sources > Add data source > Tempo
URL: http://tempo.fury-monitoring.svc.cluster.local:3200
Save & Test
```

### Interludio: Vault se sella solo

Mientras retomaba el trabajo después de unos días, entrar a `vault.fury.local` tiraba un `503 no available server` de Traefik — no un problema de DNS ni de Traefik en sí (se descartó con ping y curl directo con el header `Host`), sino el síntoma de que el Service de destino no tenía ningún endpoint sano.

```bash
kubectl exec -n fury-infra vault-0 -- vault status
# Sealed: true
```

`rathma` se había reiniciado, y con seal type Shamir simple (una sola key share, sin auto-unseal), Vault queda sellado automáticamente en cada arranque del pod — y mientras está sellado, su readiness probe falla, así que el Service se queda sin endpoints.

```bash
kubectl exec -n fury-infra vault-0 -- vault operator unseal \
  $(cat ~/secrets/vault-init.json | jq -r '.unseal_keys_b64[0]')
```

Esto no es un bug, es el comportamiento esperado del seal por default. Para producción real se resolvería con auto-unseal vía KMS externo o un seal `transit`; para este homelab esa complejidad extra no se justifica todavía, así que queda como trade-off consciente — cada reboot de `rathma` pide este comando a mano antes de que Vault vuelva a andar.

## Instrumentando gostalgia con OpenTelemetry

```go
// internal/infra/tracing/tracing.go
exporter, err := otlptracegrpc.New(ctx,
    otlptracegrpc.WithEndpoint(endpoint),
    otlptracegrpc.WithInsecure(),
)
tp := sdktrace.NewTracerProvider(
    sdktrace.WithBatcher(exporter),
    sdktrace.WithResource(resource.NewWithAttributes(
        semconv.SchemaURL,
        semconv.ServiceName("gostalgia-api"),
    )),
)
otel.SetTracerProvider(tp)
```

El init es no-fatal a propósito: si Tempo está caído o el endpoint no responde, la app loguea el error y sigue funcionando sin tracing en vez de crashear — observabilidad no debería ser un punto único de falla para el servicio que observa. El middleware de `otelgin` genera un span automático por cada request, sin tener que instrumentar cada handler a mano:

```go
r.Use(otelgin.Middleware("gostalgia-api"))
```

Un detalle de higiene que vale la pena mencionar: el primer intento de agregar las dependencias de OTel fue con `go get paquete@latest`, y eso arrastró un bump no relacionado de todo el árbol transitivo — gRPC, quic-go, el driver de Mongo — un diff de 108 líneas en `go.mod`/`go.sum` para agregar tracing. Pinear cada paquete a la versión exacta que ya usaba el resto del proyecto (`v1.41.0`) bajó eso a 24 líneas, sin ningún bump colateral. Ninguna herramienta te avisa de esto, hay que mirar el diff.

## Métricas RED y el ServiceMonitor que no aparecía

> **RED**: Rate, Errors, Duration — el patrón mínimo de métricas para cualquier servicio con requests: cuántas llegan, cuántas fallan, cuánto tardan. Con eso solo ya se arma un dashboard operativo útil.

```go
httpRequestsTotal = promauto.NewCounterVec(prometheus.CounterOpts{
    Name: "gostalgia_http_requests_total",
}, []string{"method", "route", "status"})

httpRequestDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
    Name: "gostalgia_http_request_duration_seconds",
}, []string{"method", "route"})
```

El label de ruta usa `c.FullPath()` (`/files/:id`) en vez de la URL cruda (`/files/123`) — con IDs reales en el label, cada archivo distinto generaría una serie de métrica nueva, cardinalidad que crece sin límite y termina tumbando a Prometheus.

Con las métricas expuestas en `/v1/metrics`, faltaba que Prometheus las scrapeara sola, vía `ServiceMonitor`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: gostalgia-api
  namespace: fury-dev
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      app: gostalgia-api
  endpoints:
    - targetPort: 8080
      path: /v1/metrics
```

> **Issue 02**: aplicado, con el `serviceMonitorSelector` del Prometheus Operator confirmado correcto, el target seguía sin aparecer en `/api/v1/targets`. La confusión: `ServiceMonitor.spec.selector` filtra por los labels del objeto `Service` — no por el `spec.selector` que el Service usa para elegir *pods*. Son dos cosas con el mismo nombre (`selector`) resolviendo problemas distintos. El `Service` de `gostalgia-api` no tenía ningún label propio en el cluster, aunque el YAML fuente sí lo tenía — nunca se había vuelto a aplicar después de agregarlo. Fix de urgencia: `kubectl label svc gostalgia-api app=gostalgia-api`, y después reaplicar el manifiesto para que quedara sincronizado.

Con el target en `health: "up"`, el dashboard quedó armado como código — un `ConfigMap` con el JSON del dashboard adentro, con el label `grafana_dashboard: "1"` que el sidecar de Grafana (`grafana-sc-dashboard`, viene con `kube-prometheus-stack`) descubre solo, sin reiniciar ningún pod. Mismo espíritu que el resto de esta serie: nada de clickear en una UI para dejar algo que después no se puede versionar.

![dashboard RED de gostalgia-api con tráfico real, incluyendo el pico de error rate de /v1/debug/fail](../img/2026-08-13-poor-mans-fury-part-03/gostalgia-red-dashboard.png)

## Correlación trace - logs

Con traces y métricas ya andando, faltaba la pieza que las conecta: poder pasar de un trace roto en Tempo a las líneas de log exactas de esa request, sin tocar `kubectl logs` a mano.

```go
// internal/infra/logging/logging.go
func (h *TraceHandler) Handle(ctx context.Context, r slog.Record) error {
    if span := trace.SpanContextFromContext(ctx); span.IsValid() {
        r.AddAttrs(
            slog.String("trace_id", span.TraceID().String()),
            slog.String("span_id", span.SpanID().String()),
        )
    }
    return h.Handler.Handle(ctx, r)
}
```

Un wrapper de `slog.Handler` que lee el span activo del contexto y le suma `trace_id`/`span_id` a cada línea de log, sin tocar cada call site — cualquier log hecho con las variantes `Context` de `slog` queda correlacionado solo. Un middleware nuevo loguea cualquier respuesta `>=500` con ese mismo contexto (antes no había ningún logging de errores a nivel request), y un endpoint de diagnóstico (`GET /v1/debug/fail`) devuelve un 500 a pedido, solo para poder probar el circuito completo sin esperar una falla real.

> **Issue 03**: la config de "Trace to logs" del datasource de Tempo (que conecta con Loki) estaba vacía en Grafana — nunca había quedado guardada la primera vez. Sin eso, el botón "Logs for this span" no aparece en el detalle de un trace. Reconfigurar (`Data source: Loki`, "Filter by Trace ID" activado) lo resolvió — con el `trace_id` como texto plano en cada log JSON, Loki lo puede buscar directo, sin mapear tags de span a labels.

Con eso: `curl /v1/debug/fail` → el `trace_id` aparece en el log (`kubectl logs | grep "request failed"`) → pegado en Grafana Explore (Tempo) abre el trace → el botón de logs salta directo a la línea correlacionada en Loki. De un error a la causa raíz, todo desde una sola pantalla.

![trace de gostalgia-api](../img/2026-08-13-poor-mans-fury-part-03/tempo-logs-for-span.png)

## SLOs con error budget de verdad

Esta es la parte que más se parece a lo que hacía en Fury. No "una alerta si la latencia sube", sino un objetivo real con un presupuesto de margen y una forma de detectar cuándo se está gastando ese margen demasiado rápido.

**SLI**: fracción de requests con duración `<= 200ms`. **SLO**: 99% en una ventana de 30 días. **Error budget**: el 1% restante — el margen que existe para deploys, picos de carga, degradaciones puntuales, sin que cada request lento dispare pánico.

Antes de la parte de alertas, un detalle que casi se pasa por alto: los buckets del histograma que ya existían (`prometheus.DefBuckets`) saltan de `0.1` a `0.25` — no hay ningún bucket en `0.2`. `histogram_quantile` y los ratios por bucket solo son exactos en un límite que exista como bucket real; sin uno en `0.2`, el 99%-bajo-200ms hubiera salido interpolado, y mal. Buckets custom con `0.2` explícito, alineados al SLO desde el diseño de la métrica, no después.

```go
Buckets: []float64{0.01, 0.025, 0.05, 0.1, 0.2, 0.3, 0.5, 1, 2.5, 5, 10}
```

> **Burn rate**: la velocidad a la que se gasta el error budget, comparada con el ritmo sostenible. `burn_rate = 1`, se gasta el presupuesto justo a los 30 días. Más que eso, se agota antes. Es la base de las alertas multi-window que recomienda el [SRE workbook de Google](https://sre.google/workbook/alerting-on-slos/): en vez de una sola alerta de umbral fijo, dos alertas que miran la misma métrica a distintas velocidades.

| Alerta | Ventana larga | Ventana corta | Burn rate | Budget consumido | Se agota en | Severidad |
|---|---|---|---|---|---|---|
| Fast burn | 1h | 5m | > 14.4 | 2% | ~2 días | page |
| Slow burn | 6h | 30m | > 6 | 5% | ~5 días | ticket |

Cada alerta exige que **ambas** ventanas superen el umbral a la vez — un pico de dos minutos que se resuelve solo no dispara un page. Todo esto vive en un `PrometheusRule`, recording rules más las dos alertas, mismo patrón declarativo que el `ServiceMonitor`.

Con tráfico bajo/sintético como el de este homelab, las ventanas cortas van a tener ratios ruidosos — pocas muestras por minuto. El valor acá es la técnica, no la significancia estadística de una alerta en particular.

## Dónde quedamos

`gostalgia-api` ahora se puede ver a sí misma de punta a punta: traces en Tempo, métricas RED con dashboard propio, logs correlacionados por `trace_id`, y un SLO real con alertas de burn rate en vez de un umbral suelto. El pipeline que lo despliega, además, corre en minutos.

Lo que sigue, según el plan que arrancó todo esto: usar este mismo stack para debug de sistemas distribuidos de verdad — inyectar fallas reales en `gostalgia`, no un endpoint de demo, y ver si la instrumentación que se armó acá alcanza para diagnosticarlas sin adivinar. Backstage sigue esperando, pero ahora hay algo real para catalogar.
