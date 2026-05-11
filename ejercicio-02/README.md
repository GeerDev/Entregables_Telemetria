# Ejercicio 2 — Exporters, Recording Rules y Alert Rules en Prometheus

---

## Exporters

Un **exporter** es un programa que recoge métricas de un sistema externo (que no expone métricas en formato Prometheus de forma nativa) y las convierte y expone en el formato de texto que Prometheus puede scrapear y consumir.

### ¿Por qué son necesarios?

Prometheus espera que los targets expongan un endpoint HTTP `/metrics` con un formato concreto. Muchos sistemas (bases de datos, hardware, aplicaciones legacy, etc.) no hacen eso por defecto, así que el exporter actúa de **adaptador o traductor**.

### Cómo funciona

```
Sistema externo  ──────►  Exporter  ──(HTTP /metrics)──►  Prometheus
```

1. El exporter se conecta al sistema externo para leer sus métricas internas.
2. Transforma esos datos al formato de texto Prometheus.
3. Los expone en un endpoint HTTP.
4. Prometheus scrapea ese endpoint periódicamente.

### Ejemplos habituales

| Exporter | Sistema monitorizado |
|---|---|
| `node_exporter` | Métricas del sistema operativo (CPU, RAM, disco, red) |
| `blackbox_exporter` | Comprobaciones externas (HTTP, TCP, ICMP, DNS) |
| `mysqld_exporter` | MySQL / MariaDB |
| `postgres_exporter` | PostgreSQL |
| `redis_exporter` | Redis |
| `kafka_exporter` | Apache Kafka |
| `jmx_exporter` | Aplicaciones Java (vía JMX) |

---

## Recording Rules

Las **recording rules** permiten **precalcular expresiones PromQL complejas o costosas** y guardar el resultado como una nueva serie temporal. Prometheus evalúa estas reglas periódicamente y almacena el resultado como si fuera una métrica más.

### ¿Por qué usarlas?

- Las queries PromQL sobre grandes volúmenes de datos pueden ser lentas.
- Si un dashboard o una alert rule usa la misma query repetidamente, es más eficiente precalcularla.
- Simplifican las queries en dashboards y alertas al reemplazar expresiones largas por un nombre de métrica legible.

### Estructura en el fichero de reglas

```yaml
groups:
  - name: ejemplo_recording_rules
    interval: 1m          # frecuencia de evaluación (opcional, hereda global)
    rules:
      - record: job:http_requests_total:rate5m   # nombre de la nueva métrica
        expr: rate(http_requests_total[5m])      # expresión PromQL a precalcular
```

### Convención de nombrado

El nombre suele seguir el patrón: `<nivel>:<métrica_base>:<operación>`.  
Ejemplo: `job:http_requests_total:rate5m` indica nivel `job`, métrica base `http_requests_total` y operación `rate` con ventana de `5m`.

### Uso posterior

Una vez calculada, la métrica `job:http_requests_total:rate5m` puede usarse en queries, dashboards o alert rules como cualquier otra métrica normal, pero con un coste de evaluación mínimo.

---

## Alert Rules

Las **alert rules** definen condiciones sobre métricas de Prometheus que, cuando se cumplen durante un tiempo determinado, disparan una **alerta**. Prometheus evalúa estas condiciones periódicamente y envía las alertas al **Alertmanager**, que se encarga de enrutarlas, agruparlas y notificarlas (email, Slack, PagerDuty, etc.).

### Estructura en el fichero de reglas

```yaml
groups:
  - name: ejemplo_alert_rules
    rules:
      - alert: HighCPUUsage                              # nombre de la alerta
        expr: rate(process_cpu_seconds_total[1m]) > 0.8  # condición PromQL
        for: 5m                                          # tiempo que debe cumplirse
        labels:
          severity: warning                              # etiquetas adicionales
        annotations:
          summary: "CPU alta en {{ $labels.instance }}"
          description: "El uso de CPU lleva más de 5 minutos por encima del 80%."
```

### Ciclo de vida de una alerta

```
INACTIVE  ──(condición cumplida)──►  PENDING  ──(tiempo `for` cumplido)──►  FIRING
              ◄──(condición deja de cumplirse)──────────────────────────────────────
```

| Estado | Significado |
|---|---|
| `INACTIVE` | La condición no se cumple. |
| `PENDING` | La condición se cumple pero aún no ha transcurrido el tiempo `for`. |
| `FIRING` | La alerta está activa y se envía al Alertmanager. |

### Parámetros clave

| Campo | Descripción |
|---|---|
| `alert` | Nombre identificativo de la alerta. |
| `expr` | Expresión PromQL cuyo resultado activa la alerta cuando es verdadero (valor != 0). |
| `for` | Tiempo mínimo que la condición debe mantenerse antes de pasar a FIRING. Evita falsos positivos por picos momentáneos. |
| `labels` | Etiquetas añadidas a la alerta (p. ej., `severity: critical`). |
| `annotations` | Información descriptiva para el equipo (resumen, descripción, runbook). Permiten templates con `{{ $labels.xxx }}` y `{{ $value }}`. |

---

## Resumen comparativo entre los 3 conceptos

| Concepto | Para qué sirve | Cuándo usarlo |
|---|---|---|
| **Exporter** | Adaptar métricas de sistemas externos al formato Prometheus | Sistemas que no exponen `/metrics` nativamente |
| **Recording Rule** | Precalcular y almacenar expresiones PromQL costosas | Queries lentas, dashboards en tiempo real, reutilización |
| **Alert Rule** | Definir condiciones que generan alertas | Detección de anomalías, umbrales de rendimiento/disponibilidad |
