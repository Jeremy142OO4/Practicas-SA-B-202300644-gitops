# Runbook de observabilidad

Los comandos se ejecutan en Cloud Shell o en la laptop con acceso al cluster.
No necesitan el plugin de Argo Rollouts.

## Preparar las consultas

```bash
gcloud container clusters get-credentials p9-202300644 \
  --zone us-central1-a --project sa-202300644
kubectl config current-context
kubectl get applications -n argocd
kubectl get pods -n observability
kubectl get prometheusrule p10-slos -n observability
```

En una terminal, dejar abierto:

```bash
kubectl port-forward -n observability \
  svc/p10-monitoring-kube-promet-prometheus 9090:9090
```

En otra terminal se consultan las alertas y las reglas:

```bash
curl -s http://127.0.0.1:9090/api/v1/alerts | jq '.data.alerts'
curl -s http://127.0.0.1:9090/api/v1/rules | jq '.data.groups'
```

PrometheusRule define las reglas; no es una lista de alertas activas.
No existe un recurso `alerts` de Kubernetes en este stack.

## Disponibilidad

**Alerta:** ApiAvailabilityErrorBudgetBurn.

**Por que actuar:** el consumo supera dos veces el ritmo permitido del
presupuesto de 1%, en ventanas de 1 y 5 minutos. No es un unico 5xx aislado.

1. Consultar `slo:api_availability:burn_rate_1m` y
   `slo:api_availability:burn_rate_5m` en Prometheus/Grafana.
2. Revisar si coincide con una nueva candidata.
3. Buscar una traza de la ruta con fallo en Tempo.
4. Buscar su trace_id en Loki y confirmar ruta, pod y error.
5. Confirmar que Argo Rollouts aborto la candidata y que stable responde.
6. Si tambien falla stable, investigar la dependencia indicada en la traza;
   corregir el manifiesto mediante un PR a GitOps.

## Latencia

**Alerta:** ApiLatencyErrorBudgetBurn.

**Por que actuar:** se consume el presupuesto de solicitudes lentas (5%)
a mas de dos veces el ritmo permitido. Una latencia alta aislada no basta.

1. Consultar `slo:api_latency:burn_rate_1m` y
   `slo:api_latency:burn_rate_5m`; revisar el p95.
   La alerta se evalua tambien por pod (estable o candidato); consultar
   `slo:gateway_pod_latency:burn_rate_1m` y `slo:gateway_pod_latency:burn_rate_5m`.
2. Buscar en Tempo la ruta /orders/graphql durante el intervalo de la alerta.
3. Abrir la traza y comparar la duracion de gateway, orders-ms y catalog-ms.
4. En Loki buscar el mismo trace_id. Para la falla de demostracion,
   el evento `demo_delay` registra la demora configurada.
5. Confirmar AnalysisRun Failed y la respuesta rapida de stable.
6. Revertir la configuracion de demora mediante GitOps para que el estado
   deseado vuelva a coincidir con la version estable.

## Consultar una metrica

```bash
curl -sG http://127.0.0.1:9090/api/v1/query \
  --data-urlencode 'query=slo:api_latency:error_budget:ratio' | jq '.data.result'
kubectl get analysisruns -n sa-p9
kubectl describe rollout api-gateway -n sa-p9
kubectl get rollout api-gateway -n sa-p9 -o json | \
  jq '{phase:.status.phase, abort:.status.abort, stable:.status.stableRS}'
```

## Correlacionar traza y registro

En terminales separadas, dejar abiertos estos accesos:

```bash
kubectl port-forward -n observability svc/tempo 3200:3200
kubectl port-forward -n observability svc/loki 3100:3100
```

Sustituir ID_REAL por el identificador observado en Tempo:

```bash
curl -s -H 'Accept: application/json' \
  http://127.0.0.1:3200/api/traces/ID_REAL | jq
curl -sG http://127.0.0.1:3100/loki/api/v1/query_range \
  --data-urlencode 'query={namespace="sa-p9"} | json | trace_id="ID_REAL"' | jq
```

El tiempo del recorrido empieza al observar la alerta y termina al
identificar el registro que explica la causa. Se guarda la hora de ambos pasos.

## Abrir Grafana

```bash
kubectl port-forward --address 0.0.0.0 -n observability \
  svc/p10-monitoring-grafana 8086:80
```

En Cloud Shell usar Vista previa web, puerto 8086. Usuario: admin.
La contraseña esta en el Secret administrado por el chart; no se guarda en Git:

```bash
kubectl get secret p10-monitoring-grafana -n observability \
  -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

Abrir el tablero **SA Platform - P10**. Las alertas son de Prometheus y
Alertmanager; la portada de Grafana puede mostrar cero alertas de Grafana
aunque una regla de Prometheus este disparada.

## Recuperacion y limites

El aborto automatico de Argo Rollouts conserva stable. La reversion en GitOps
restaura posteriormente el estado deseado y ArgoCD sincroniza.
No se cambia directamente la imagen con kubectl.

Si no hay muestras, el analisis no debe promocionar. Verificar ServiceMonitor,
targets y trafico antes de reintentar. Un historial de pocos minutos no
demuestra una hora completa. Los callbacks de pago se cuentan como resultados
procesados, no como cobros unicos ni ingresos monetarios.

## Sin trafico y recoleccion unica

El gateway usa un PodMonitor: cada pod se consulta una vez, aunque los dos
Services apunten a el. El analisis identifica la candidata por su hash, no
por el Service que se uso para recoger la metrica.

Sin solicitudes de negocio en la ultima hora, el SLO no publica una proporcion:
no equivale a 0% ni a 100%. El tablero muestra Sin trafico / sin datos.
Primero comprobar targets up y luego efectuar solicitudes reales de baja
frecuencia; no generar trafico permanente solo para llenar las graficas.
