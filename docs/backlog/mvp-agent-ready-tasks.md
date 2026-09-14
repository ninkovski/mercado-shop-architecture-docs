# MVP Agent-Ready Tasks

Este backlog está escrito para que agentes o desarrolladores puedan ejecutar tareas sin ambigüedad.

## Definición general de listo

Una tarea está lista cuando tiene:
- objetivo claro
- alcance acotado
- entradas y salidas esperadas
- criterios de aceptación
- Definition of Done

## 1. Crear contrato base de API v1 para catálogo y carrito

### Objetivo
Definir los endpoints públicos mínimos para consultar productos y gestionar carrito.

### Alcance
- `GET /v1/products`
- `GET /v1/products/{productId}`
- `POST /v1/carts`
- `POST /v1/carts/{cartId}/items`
- `GET /v1/carts/{cartId}`

### Criterios de aceptación
- Los endpoints están documentados.
- Cada request/response tiene ejemplo JSON.
- La versión está explícita en la URL.
- Los errores siguen un formato común.

### Definition of Done
- Documento publicado.
- Ejemplos validados.
- Contrato listo para implementar.

## 2. Definir flujo de creación de orden

### Objetivo
Establecer cómo se pasa de carrito validado a orden `pending_payment`.

### Alcance
- validación de carrito
- validación de cobertura
- persistencia de orden
- snapshot de items

### Criterios de aceptación
- La orden guarda snapshot de precio y cantidad.
- La orden incluye `tenantId`.
- La orden tiene estado inicial definido.

### Definition of Done
- Flujo documentado end-to-end.
- Estados de orden definidos.
- Errores funcionales listados.

## 3. Diseñar contrato de checkout con pagos

### Objetivo
Definir cómo se inicia el pago desde backend.

### Alcance
- entrada de checkout
- idempotency key
- respuesta con URL o token de checkout
- integración desacoplada con proveedor

### Criterios de aceptación
- El request exige `Idempotency-Key`.
- La respuesta devuelve `paymentIntentId`.
- El proveedor de pagos no está acoplado al dominio.

### Definition of Done
- Contrato documentado.
- Campos mínimos definidos.
- Casos de error cubiertos.

## 4. Diseñar webhook de confirmación de pago

### Objetivo
Procesar eventos del proveedor de pago sin duplicar efectos.

### Alcance
- firma de webhook
- deduplicación
- cambio de estado de orden
- publicación de `order.paid.v1`

### Criterios de aceptación
- El webhook es idempotente.
- Los duplicados no reescriben el estado.
- El evento publicado tiene envelope estándar.

### Definition of Done
- Contrato del webhook documentado.
- Estados y respuestas definidos.
- Reintentos y DLQ considerados.

## 5. Definir servicio de notificación post-pago

### Objetivo
Enviar email después de confirmar el pago.

### Alcance
- consumir `order.paid.v1`
- preparar plantilla
- enviar email
- registrar resultado

### Criterios de aceptación
- La notificación se dispara asíncronamente.
- Si falla, se reintenta.
- Si agota reintentos, va a DLQ.

### Definition of Done
- Evento de entrada definido.
- Payload de notificación documentado.
- Estrategia de error documentada.

## 6. Definir validación de cobertura geográfica

### Objetivo
Validar si una dirección puede recibir entrega.

### Alcance
- reglas por tenant
- consulta por país/estado/ciudad/código postal
- costo de envío base

### Criterios de aceptación
- La validación devuelve `valid` o `invalid`.
- El resultado incluye costo si aplica.
- La lógica se puede usar antes del checkout.

### Definition of Done
- Contrato documentado.
- Estructura de tabla definida.
- Casos límite listados.

## 7. Definir eventos de dominio v1

### Objetivo
Establecer los eventos mínimos compartidos por la plataforma.

### Alcance
- `order.created.v1`
- `order.paid.v1`
- `payment.confirmed.v1`
- `notification.email.requested.v1`

### Criterios de aceptación
- Todos usan el mismo envelope.
- Todos incluyen `eventId`, `tenantId`, `correlationId`.
- Cambios incompatibles están prohibidos sin nueva versión.

### Definition of Done
- Documentación publicada.
- Ejemplos JSON agregados.
- Reglas de compatibilidad definidas.

## 8. Definir esquema mínimo de Airtable

### Objetivo
Documentar las tablas mínimas para el MVP.

### Alcance
- tenants
- products
- carts
- cart_items
- promotions
- orders
- order_items
- payment_attempts
- coverage_zones
- notifications

### Criterios de aceptación
- Cada tabla tiene propósito claro.
- Cada tabla tiene campos mínimos.
- Las relaciones están definidas.

### Definition of Done
- Documento aprobado.
- Esquema alineado con el flujo MVP.

## 9. Definir estándares de observabilidad

### Objetivo
Asegurar trazabilidad de requests, eventos y fallos.

### Alcance
- correlation IDs
- logging estructurado
- métricas básicas
- rastreo de eventos

### Criterios de aceptación
- Todo request crítico puede rastrearse.
- Los eventos usan metadata común.
- Los errores relevantes quedan auditables.

### Definition of Done
- Guía publicada.
- Campos de observabilidad estandarizados.

## 10. Crear plantilla de repo para nuevos servicios/functions

### Objetivo
Acelerar el arranque de futuros repos de implementación.

### Alcance
- estructura de carpetas
- README base
- ejemplos de contratos
- plantilla de tests

### Criterios de aceptación
- La plantilla distingue sync, async y service.
- Incluye convenciones de naming.
- Permite iniciar un repo sin inventar estructura.

### Definition of Done
- Plantilla documentada.
- Lista para reutilización por agentes.

## Regla final del backlog

Si una tarea no puede completarse con un entregable verificable, todavía no está lista para agentes.
