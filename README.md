# Entregables Telemetria

### 1. Ejecuta Prometheus en local usando Docker

- Crea una configuración mínima que realice un 'scraping' del propio Prometheus.
- Mapea el puerto del cliente web por defecto, y realiza las siguientes queries:
  - Una cualquiera sobre memoria utilizada.
  - Una cualquiera sobre cpu utilizada.

➡️ [Ver solución Ejercicio 1](ejercicio-01/README.md)

### 2. Explica que son los exporters, recording rules y alert rules en el contexto de Prometheus

➡️ [Ver solución Ejercicio 2](ejercicio-02/README.md)

### 3. Explica el contenido de la carpeta [01-start-up-loki](https://github.com/Lemoncode/bootcamp-devops-lemoncode/tree/feature/iac-review/06-monitoring/01-exercise/01-start-up-loki)

➡️ [Ver solución Ejercicio 3](ejercicio-03/README.md)

### 4. Explica la estructura de una traza en Jaeger

- span
- traza
- scope
- tags

➡️ [Ver solución Ejercicio 4](ejercicio-04/README.md)

### 5. Desafíos

#### 5.1 Crea un docker compose para realizar un setup con Prometheus.
- Debe incluir como servicio del docker compose la siguiente [App](https://github.com/JaimeSalas/non-political-map/tree/main/app_map).
- Instala la librería [cliente de Python](https://github.com/prometheus/client_python) para extraer métricas por defecto.
- Genera un servicio Prometheus que como 'target' tenga la app anterior.
- Verifica que el 'target' es alcanzado por Prometheus.

**NOTA**: Para instalar la librería seguir los pasos [aquí](https://prometheus.github.io/client_python/exporting/http/fastapi-gunicorn/) detallados.

➡️ [Ver solución Ejercicio 5.1](ejercicio-05-1/README.md)

#### 5.2 Desafío Jaeger
El siguiente [tutoríal](https://medium.com/opentracing/take-opentracing-for-a-hotrod-ride-f6e3141f7941) muestra cómo usar Jaeger, sigue los pasos para solucionar los distintos issues y crea notas con tu experiencia.

➡️ [Ver solución Ejercicio 5.2](ejercicio-05-2/README.md)
