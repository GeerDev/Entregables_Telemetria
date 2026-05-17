# Ejercicio 5.2 — Desafío Jaeger: Tutorial HotROD

## Descripción del tutorial

**HotROD** (Rides on Demand) es una aplicación de ejemplo de Uber que simula un servicio de transporte bajo demanda. Está diseñada específicamente para demostrar las capacidades del trazado distribuido con Jaeger y mostrar cómo diagnosticar y solucionar problemas de rendimiento usando trazas.

La aplicación tiene múltiples servicios (frontend, customer, driver, route) que se comunican entre sí, con varios problemas de rendimiento intencionalmente introducidos que se pueden detectar y resolver con Jaeger.

---

## Levantar el entorno

### Docker Compose

```yaml
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: jaeger
    ports:
      - "16686:16686"
      - "4317:4317"
      - "4318:4318"
    environment:
      - COLLECTOR_OTLP_ENABLED=true

  hotrod:
    image: jaegertracing/example-hotrod:latest
    container_name: hotrod
    command: ["all"]
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4318
      - OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
    ports:
      - "8080:8080"
    depends_on:
      - jaeger
```

```bash
docker compose up -d
```

---

## Acceso

| URL | Descripción |
|---|---|
| http://localhost:8080 | App HotROD |
| http://localhost:16686 | UI de Jaeger |

---

## Generando trazas

1. Abrimos **http://localhost:8080**.
2. Hacemos clic en cualquiera de los botones de pasajero para solicitar un coche.
3. La app genera una petición que atraviesa los servicios `frontend`, `customer`, `driver` y `route`.
4. Vamos a **http://localhost:16686**, seleccionamos el servicio `frontend` y buscamos las trazas recientes.

---

## Issues detectados y soluciones

### Issue 1: Mutex contention (contención de mutex)

**Síntoma detectado en Jaeger:**  
Al buscar trazas del servicio `driver`, se observa que muchos spans de `FindNearest` tienen duraciones muy altas y se solapan poco. En el log del span aparece el tag `sampler.type=ratelimiting` y hay eventos de tipo "lock acquired".

**Causa:**  
El servicio `driver` usa un `sync.Mutex` global para proteger el acceso a los datos. Cuando hay múltiples peticiones concurrentes, se produce contención: cada petición tiene que esperar a que la anterior libere el lock.

**Solución:**  
Agregar el flag `--fix-disable-db-conn-mutex` al comando de hotrod en el `docker-compose.yml`:
```yaml
command: ["all", "--fix-disable-db-conn-mutex"]
```

---

### Issue 2: N+1 queries (consultas en bucle)

**Síntoma detectado en Jaeger:**  
En la traza del servicio `customer`, se ve un span padre `GetCustomer` que contiene decenas de spans hijos idénticos llamados `SQL SELECT`. Cada uno dura pocos milisegundos, pero la suma hace que la operación total tarde mucho.

**Causa:**  
El código carga el cliente y luego hace una consulta SQL individual por cada campo relacionado (patrón N+1). En lugar de hacer una única query con JOIN, hace N queries adicionales.

**Solución:**  
Agregar el flag `--fix-db-query-delay=0ms` para eliminar el delay artificial:
```yaml
command: ["all", "--fix-db-query-delay=0ms"]
```

---

### Issue 3: Missing cache (caché no utilizada)

**Síntoma detectado en Jaeger:**  
Las trazas de peticiones del mismo cliente muestran el mismo tiempo de respuesta elevado en todas las llamadas. No se observa ninguna diferencia entre la primera llamada y las siguientes, lo que indica que no hay caché funcionando.

**Causa:**  
El servicio tiene código de caché implementado pero con un bug: la clave de caché siempre se genera de forma incorrecta (por ejemplo, incluye un timestamp), haciendo que nunca haya un cache hit.

**Solución:**  
Agregar Redis al `docker-compose.yml` y configurar la variable `REDIS_HOST`:
```yaml
services:
  redis:
    image: redis:7-alpine
    container_name: redis

  hotrod:
    environment:
      - REDIS_HOST=redis:6379
    depends_on:
      - jaeger
      - redis
```

---

### Issue 4: Baggage — propagación de contexto excesiva

**Síntoma detectado en Jaeger:**  
Al revisar los tags de los spans, se observa que hay datos de gran tamaño propagados como **baggage** (contexto de traza). Esto se refleja en latencias adicionales en cada llamada HTTP entre servicios.

**Causa:**  
El baggage viaja en las cabeceras HTTP de cada petición. Si se almacena información grande (como datos del cliente completo), aumenta el overhead de red en cada hop.

**Solución:**  
Agregar el flag `--fix-route-worker-pool-size` para aumentar el pool de workers y reducir la serialización:
```yaml
command: ["all", "--fix-route-worker-pool-size=10"]
```

---

## Observaciones generales

- Jaeger permite ver de forma inmediata **dónde se pierde el tiempo** dentro de una traza sin necesidad de logs adicionales.
- Los **tags** (`error=true`, `http.status_code`, `db.statement`) son fundamentales para correlacionar síntomas con causas.
- La vista de **comparación de trazas** (Compare) es muy útil para contrastar el comportamiento antes y después de un fix.
- La integración con **Grafana** permite combinar métricas de Prometheus con trazas de Jaeger para una observabilidad completa.

---

## Resumen de causas — qué mirar en cualquier sistema

| Issue | Señal en Jaeger | Causa raíz | Fix |
|---|---|---|---|
| **Mutex contention** | Spans del mismo tipo en serie, nunca solapados | Lock global que serializa peticiones concurrentes | `--fix-disable-db-conn-mutex` |
| **N+1 queries** | Span padre con decenas de spans hijos idénticos | Bucle que hace una llamada I/O por cada elemento | `--fix-db-query-delay=0ms` |
| **Missing cache** | Latencia idéntica en llamadas repetidas, `error=true` en spans de Redis | Caché no disponible o con clave incorrecta | Agregar Redis + `REDIS_HOST` |
| **Worker pool exhaustion** | Gap grande entre inicio del span padre y el primer hijo | Pool de workers demasiado pequeño, peticiones encoladas | `--fix-route-worker-pool-size=10` |

**Regla general:**
- Si `duración total > suma de spans hijos` → hay tiempo perdido esperando (lock, pool, red)
- Si hay muchos spans hijos con el mismo nombre → hay un bucle que debería ser una llamada batch
