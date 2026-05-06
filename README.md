# Entregables Telemetria

### 1. Ejecuta Prometheus en local usando Docker

- Crea una configuración mínima que realice un 'scraping' del propio Prometheus.
- Mapea el puerto del cliente web por defecto, y realiza las siguientes queries:
  - Una cualquiera sobre memoria utilizada.
  - Una cualquiera sobre cpu utilizada.

### 2. Explica que son los exporters, recording rules y alert rules en el contexto de Prometheus
### 3. Explica el contenido de la carpeta 01-start-up-loki
### 4. Explica la estructura de una traza en Jaeger

- span
- traza
- scope
- tags

### 5. Desafíos

#### 5.1 Crea un docker compose para realizar un setup con Prometheus.
- Debe incluir como servicio del docker compose la siguiente App.
- Instala la librería cliente de Python para extraer métricas por defecto.
- Genera usn servicio Prometheus que como 'target' tenga la app anterior.
- Verifica que el 'target' es alcanzado por Prometheus.

**NOTA**: Para instalar la librería seguir los pasos aquí detallados.

#### 5.2 Desafío Jaeger
El siguiente tutoríal muestra cómo usar Jaeger, sigue los pasos para solucionar los distintos issues y crea notas con tu experiencia.
