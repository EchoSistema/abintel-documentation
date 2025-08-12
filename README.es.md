# Resumen de la API AbIntel

AbIntel ofrece microservicios plug-and-play que convierten transacciones, facturas, tickets, flujos de sensores o texto libre en predicciones en menos de un minuto, permitiendo iniciar proyectos sin un equipo de ML dedicado.

## Panorama de la Plataforma
* Escalado automático en Kubernetes hasta 32 T4, con soporte opcional de H100
* Auditoría SOC 2 Tipo 2 en progreso y centros de datos certificados ISO 27001
* Despliegues canario sin tiempo de inactividad
* Versionado semántico mediante `/v1`

URL base de la API: `https://api-prod.abintel.app`

## Autenticación
Cada petición debe incluir:
- `X-Customer-Api-Id`: UUID del cliente
- `X-Secret`: secreto de 64 caracteres rotado cada 90 días
- `Accept-Language`: idioma de la respuesta (`en`, `es`, `pt`)

## Encolar una Tarea
`POST /api/v1/ai/{route}` encola un trabajo de IA. Ejemplo:
```
POST /api/v1/ai/forecast_revenue
```
El cuerpo puede contener cualquier JSON y se almacena tal como fue enviado. El servicio guarda la tarea en Cosmos DB, descuenta un `UsageRecord` y envía el trabajo a Azure Service Bus. Los workers ejecutan el modelo y actualizan el estado al terminar.

## Consultar el Estado
- `GET /api/v1/taskadmin/tasks?status_filter=completed&limit=20`
- `GET /api/v1/taskadmin/tasks/{task_id}?part=response`

Ambos usan los mismos encabezados de autenticación y devuelven la salida del modelo o un mensaje de error.

## Paneles y Uso
- `GET /api/v1/taskadmin/queue/stats`
- `GET /api/v1/taskadmin/queue/queued?queue=medium`
- `GET /api/v1/taskadmin/usage/month`
- `GET /api/v1/taskadmin/usage/history`

Estos endpoints consultan Cosmos DB para mostrar el estado de las colas y seguir las llamadas y costos mensuales.

## Ciclo Típico
1. El cliente envía el payload a `/ai/...`
2. El proxy guarda y encola la tarea
3. El worker procesa el trabajo y guarda la respuesta
4. El cliente obtiene el resultado mediante `/taskadmin/tasks/{id}?part=response`

Los estados comunes son `queued`, `completed` y `failed`.

## Catálogo de Rutas
- **Customer Intelligence**: segmentación de compras, fidelidad, riesgo de churn y más
- **Propensity & Recommendation**: propensión de compra/upgrade, recomendaciones, cross-sell, uplift modeling
- **Forecasting & Planning**: pronósticos de ingresos y unidades, proyecciones de costos
- **Inventory & Supply-Chain**: optimización, análisis de exceso de inventario, informes NLP
- **Cost & Risk Analytics**: riesgo crediticio, atribución de canal, detección de anomalías, análisis de sentimiento

