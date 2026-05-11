# Ejercicio 4 — Estructura de una traza en Jaeger

---

## Contexto

Jaeger es un sistema de **trazado distribuido** (distributed tracing). Permite seguir el camino de una petición a través de múltiples servicios de una arquitectura de microservicios, medir cuánto tarda cada paso y detectar dónde ocurren errores o cuellos de botella.

---

## Traza (Trace)

Una **traza** representa el **recorrido completo de una petición** a través del sistema, desde que entra hasta que sale. Agrupa todos los pasos individuales que se ejecutaron para satisfacer esa petición, independientemente de en cuántos servicios o procesos distintos ocurrieron.

### Características

- Identificada por un **Trace ID** único (16 bytes en hexadecimal), que se propaga a lo largo de toda la cadena de llamadas mediante cabeceras HTTP (p. ej., `X-B3-TraceId` o `traceparent` en W3C Trace Context).
- Está formada por uno o más **spans** relacionados entre sí en una jerarquía padre-hijo.
- Tiene una duración total que abarca desde el primer span hasta el último.

### Representación visual

En la UI de Jaeger, una traza se muestra como un **diagrama de Gantt** donde cada fila es un span y su posición horizontal indica cuándo empezó y cuánto duró.

```
Trace ID: abc123...
│
├── [frontend]      GET /checkout           0ms ───────────────────── 350ms
│   ├── [order-svc] createOrder             10ms ────────── 200ms
│   │   └── [db]    INSERT orders           15ms ─── 50ms
│   └── [payment]   chargeCard             210ms ──────── 130ms
```

---

## Span

Un **span** es la **unidad mínima de trabajo** dentro de una traza. Representa una operación concreta: una llamada HTTP, una consulta a base de datos, un procesamiento interno, etc.

### Campos principales de un span

| Campo | Descripción |
|---|---|
| **Span ID** | Identificador único del span dentro de la traza. |
| **Trace ID** | ID de la traza a la que pertenece. |
| **Parent Span ID** | ID del span padre (vacío si es el span raíz). |
| **Operation Name** | Nombre de la operación (p. ej., `HTTP GET /users`, `db.query`). |
| **Start Time** | Marca de tiempo de inicio (timestamp en microsegundos). |
| **Duration** | Duración de la operación. |
| **Tags** | Metadatos clave-valor sobre el span (ver sección Tags). |
| **Logs** | Eventos puntuales ocurridos durante el span (con timestamp propio). |
| **References** | Relación con otros spans: `ChildOf` (llamada síncrona) o `FollowsFrom` (llamada asíncrona). |
| **Baggage** | Datos propagados a lo largo de toda la traza (uso limitado por overhead). |

### Jerarquía de spans

Los spans se organizan en árbol. El **span raíz** no tiene padre y representa el punto de entrada de la petición. Los demás son hijos de quien los creó:

```
Span raíz (frontend)
├── Span hijo (order-service)
│   └── Span nieto (database)
└── Span hijo (payment-service)
```

---

## Scope (Ámbito de instrumentación)

En el contexto de **OpenTelemetry** (estándar sobre el que se basa la instrumentación moderna compatible con Jaeger), un **scope** (también llamado *Instrumentation Scope* o *Instrumentation Library*) identifica **qué librería o componente generó la telemetría**.

### Qué contiene un scope

| Campo | Descripción |
|---|---|
| **Name** | Nombre de la librería de instrumentación (p. ej., `opentelemetry-fastapi`, `opentelemetry-requests`). |
| **Version** | Versión de la librería instrumentada. |
| **Schema URL** | URL del schema semántico de atributos utilizado (opcional). |

### Para qué sirve

- Permite saber **qué generó cada span**: instrumentación automática de un framework, instrumentación manual del desarrollador, una librería cliente, etc.
- Facilita el filtrado y la gestión de la telemetría: se puede deshabilitar o configurar la instrumentación a nivel de scope.
- En la UI de Jaeger aparece reflejado en la columna **Service** junto al nombre del proceso.

### Ejemplo

```
Scope: opentelemetry-instrumentation-fastapi v0.46b0
  └── Span: GET /items
Scope: opentelemetry-instrumentation-sqlalchemy v0.46b0
  └── Span: SELECT items
```

---

## Tags

Los **tags** son **pares clave-valor** que añaden contexto y metadatos a un span. Describen el entorno, el resultado y las características de la operación representada.

### Tipos de valores

Los valores pueden ser `string`, `bool` o `number`.

### Tags estándar (convenciones semánticas de OpenTelemetry)

| Tag | Tipo | Descripción |
|---|---|---|
| `http.method` | string | Método HTTP (`GET`, `POST`, etc.) |
| `http.url` | string | URL completa de la petición |
| `http.status_code` | int | Código de respuesta HTTP |
| `db.system` | string | Sistema de base de datos (`postgresql`, `redis`, etc.) |
| `db.statement` | string | Sentencia SQL ejecutada |
| `error` | bool | `true` si el span terminó con error |
| `span.kind` | string | Rol del span: `client`, `server`, `producer`, `consumer`, `internal` |
| `net.peer.ip` | string | IP del servicio remoto |
| `net.peer.port` | int | Puerto del servicio remoto |

### Tags de proceso/servicio

Además de los tags del span, Jaeger almacena tags a nivel de **proceso** (el servicio que generó el span):

| Tag | Descripción |
|---|---|
| `service.name` | Nombre del servicio |
| `service.version` | Versión del servicio |
| `host.name` | Nombre del host |
| `telemetry.sdk.name` | SDK de telemetría utilizado |

### Cómo se usan en Jaeger

En la UI de Jaeger los tags se pueden usar para **buscar y filtrar trazas**:

```
Service: frontend
Operation: GET /checkout
Tags: http.status_code=500 error=true
```

---

## Resumen visual

```
TRACE (abc123...)
│  Duración total: 350ms
│  Timestamp inicio: 2026-05-08T10:00:00Z
│
├── SPAN: GET /checkout  [frontend]  0–350ms
│   │  Scope: opentelemetry-fastapi v0.46
│   │  Tags: http.method=GET, http.status_code=200, span.kind=server
│   │
│   ├── SPAN: createOrder  [order-service]  10–210ms
│   │   │  Scope: manual instrumentation
│   │   │  Tags: span.kind=internal
│   │   │
│   │   └── SPAN: INSERT orders  [postgres]  15–65ms
│   │          Scope: opentelemetry-sqlalchemy v0.46
│   │          Tags: db.system=postgresql, db.statement="INSERT INTO orders..."
│   │
│   └── SPAN: chargeCard  [payment-service]  210–340ms
│          Scope: opentelemetry-requests v0.46
│          Tags: http.method=POST, http.url=https://payments.api/charge, span.kind=client
```
