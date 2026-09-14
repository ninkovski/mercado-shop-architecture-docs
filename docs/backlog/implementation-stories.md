# Historias de implementación (agent-ready)

Este documento define historias de implementación listas para que un agente construya componentes MVP sin ambigüedad.

## Estructura sugerida para backlog por dominio

```text
docs/backlog/
├── mvp-agent-ready-tasks.md
└── implementation-stories.md
```

## Reglas transversales para todas las historias

- `tenantId` es obligatorio en requests, respuestas y eventos de negocio.
- Contratos HTTP y eventos deben salir de `ms-shared-contracts`.
- APIs síncronas: request/response corto y validación inmediata.
- Procesos asíncronos: idempotencia, reintentos y DLQ.
- Foco MVP: resolver flujo de venta/pago/notificación sin sobreingeniería.

---

## Historia 1: `commerce-sync-fn`

### Título
Implementar API síncrona de commerce para catálogo, carrito, promociones y creación de órdenes.

### Objetivo
Exponer endpoints `/v1` que permitan navegar catálogo, operar carrito y crear orden `pending_payment` con snapshot de precios.

### Alcance
- `GET /v1/products`
- `GET /v1/products/{productId}`
- `POST /v1/carts`
- `POST /v1/carts/{cartId}/items`
- `GET /v1/carts/{cartId}`
- `POST /v1/orders` (crea orden desde carrito)
- Aplicar promociones vigentes en cálculo del total.

### Inputs / Outputs

**Input ejemplo: `POST /v1/orders`**
```json
{
  "tenantId": "tenant_001",
  "cartId": "cart_123",
  "customer": {
    "email": "cliente@ejemplo.com"
  },
  "shippingAddress": {
    "country": "AR",
    "state": "CABA",
    "city": "Buenos Aires",
    "postalCode": "C1001"
  },
  "idempotencyKey": "idem-order-001"
}
```

**Output ejemplo: `201 Created`**
```json
{
  "tenantId": "tenant_001",
  "orderId": "ord_123",
  "status": "pending_payment",
  "currency": "ARS",
  "subtotal": 15000,
  "discountTotal": 1000,
  "shippingCost": 1990,
  "total": 15990,
  "createdAt": "2026-09-14T03:00:00Z"
}
```

### Dependencias
- `ms-shared-contracts` (DTOs y errores estándar)
- `coverage-sync-fn` (validar cobertura/costo antes de crear orden)
- Airtable (`products`, `carts`, `cart_items`, `promotions`, `orders`, `order_items`)

### Criterios de aceptación
- Todos los endpoints exigen `tenantId`.
- `POST /v1/orders` es idempotente por `idempotencyKey`.
- La orden persiste snapshot de items/precios/promoción.
- Errores funcionales usan formato estándar (`code`, `message`, `details`).

### Definition of Done
- Endpoints implementados y documentados con ejemplos JSON.
- Validaciones mínimas (campos requeridos, cantidades > 0, carrito no vacío).
- Tests de casos principales (happy path + error de validación).
- Logs estructurados con `tenantId` y `correlationId`.

### Prompt para agente (copiar/pegar)
```text
Implementa `commerce-sync-fn` en modo MVP con endpoints `/v1` para catálogo, carrito, promociones y creación de órdenes.

Requisitos obligatorios:
1) `tenantId` obligatorio en todos los flujos.
2) Endpoints: GET /v1/products, GET /v1/products/{productId}, POST /v1/carts, POST /v1/carts/{cartId}/items, GET /v1/carts/{cartId}, POST /v1/orders.
3) `POST /v1/orders` debe:
   - validar carrito
   - consultar cobertura (sync) para costo de envío
   - aplicar promociones activas
   - crear orden `pending_payment`
   - guardar snapshot de ítems y precios
   - ser idempotente con `idempotencyKey`
4) Usar contratos de `ms-shared-contracts`.
5) Manejo de errores homogéneo y logs con `tenantId` + `correlationId`.
6) Implementar tests de happy path y validaciones clave.

No agregues sobreingeniería ni nuevas capacidades fuera del alcance MVP.
```

---

## Historia 2: `coverage-sync-fn`

### Título
Implementar validación geográfica y cálculo de costo de envío en API síncrona.

### Objetivo
Responder rápidamente si una dirección está cubierta por `tenantId` y devolver costo estimado de envío.

### Alcance
- `POST /v1/coverage/validate`
- Evaluación por país/estado/ciudad/código postal.
- Devolver `isCovered` y `shippingCost`.

### Inputs / Outputs

**Input ejemplo**
```json
{
  "tenantId": "tenant_001",
  "address": {
    "country": "AR",
    "state": "CABA",
    "city": "Buenos Aires",
    "postalCode": "C1001"
  }
}
```

**Output ejemplo cubierto**
```json
{
  "tenantId": "tenant_001",
  "isCovered": true,
  "shippingCost": 1990,
  "currency": "ARS",
  "coverageZoneId": "zone_caba_01"
}
```

**Output ejemplo no cubierto**
```json
{
  "tenantId": "tenant_001",
  "isCovered": false,
  "shippingCost": null,
  "reason": "OUT_OF_COVERAGE"
}
```

### Dependencias
- `ms-shared-contracts`
- Airtable (`coverage_zones`)

### Criterios de aceptación
- Respuesta determinística para la misma dirección/tenant.
- Si no hay cobertura, no devuelve costo.
- Timeouts y errores técnicos retornan error estándar.

### Definition of Done
- Endpoint implementado con validaciones de input.
- Tests de cobertura positiva y negativa.
- Documentación de reglas mínimas de matching geográfico.

### Prompt para agente (copiar/pegar)
```text
Implementa `coverage-sync-fn` con endpoint POST /v1/coverage/validate.

Requisitos:
1) Request con `tenantId` y address (country, state, city, postalCode).
2) Responder `isCovered` y `shippingCost` cuando aplique.
3) Leer reglas por tenant desde `coverage_zones` (Airtable).
4) Contratos en `ms-shared-contracts`.
5) Errores en formato estándar y logs con `tenantId`/`correlationId`.
6) Tests mínimos: dirección cubierta, no cubierta, input inválido.

Mantener solución MVP simple y síncrona.
```

---

## Historia 3: `payments-sync-fn`

### Título
Implementar checkout síncrono y creación de intento de pago.

### Objetivo
Permitir iniciar pago de una orden `pending_payment` devolviendo `paymentIntentId` y URL/token de checkout.

### Alcance
- `POST /v1/payments/checkout`
- Crear intento de pago interno y delegar al `ms-payments-service`.
- Exigir idempotencia para evitar intentos duplicados.

### Inputs / Outputs

**Input ejemplo**
```json
{
  "tenantId": "tenant_001",
  "orderId": "ord_123",
  "amount": 15990,
  "currency": "ARS",
  "idempotencyKey": "idem-pay-001",
  "returnUrl": "https://shop.example.com/checkout/result"
}
```

**Output ejemplo**
```json
{
  "tenantId": "tenant_001",
  "orderId": "ord_123",
  "paymentIntentId": "pi_789",
  "checkoutUrl": "https://provider.example/checkout/abc",
  "status": "requires_action"
}
```

### Dependencias
- `ms-payments-service`
- `ms-shared-contracts`
- Airtable (`orders`, `payment_attempts`)

### Criterios de aceptación
- Solo permite checkout para órdenes `pending_payment`.
- `idempotencyKey` evita crear múltiples intentos equivalentes.
- No expone datos sensibles del proveedor en la respuesta.

### Definition of Done
- Endpoint implementado y documentado.
- Tests: orden válida, orden inválida, idempotencia.
- Persistencia de intento de pago con trazabilidad.

### Prompt para agente (copiar/pegar)
```text
Implementa `payments-sync-fn` para iniciar checkout.

Requisitos:
1) Endpoint POST /v1/payments/checkout.
2) Validar que la orden exista, pertenezca al `tenantId` y esté en `pending_payment`.
3) Crear `payment_attempt` interno y llamar a `ms-payments-service`.
4) Requerir `idempotencyKey` y hacer operación idempotente.
5) Responder con `paymentIntentId`, `checkoutUrl` y estado inicial.
6) Usar contratos y errores estándar de `ms-shared-contracts`.
7) Agregar tests mínimos de flujo correcto + validaciones.

No mezclar webhook ni confirmación asíncrona en este repo.
```

---

## Historia 4: `payments-async-fn`

### Título
Implementar webhook asíncrono de pagos con deduplicación y emisión de `order.paid.v1`.

### Objetivo
Procesar confirmaciones del proveedor de pago de forma idempotente y emitir evento de dominio para continuidad del flujo.

### Alcance
- Endpoint webhook (async): `POST /v1/payments/webhooks/provider`
- Verificación de firma.
- Deduplicación por `providerEventId`.
- Confirmación de pago y actualización de orden.
- Publicación de evento `order.paid.v1`.

### Inputs / Outputs

**Input webhook ejemplo**
```json
{
  "providerEventId": "pev_001",
  "type": "payment.succeeded",
  "data": {
    "paymentIntentId": "pi_789",
    "orderId": "ord_123",
    "tenantId": "tenant_001",
    "amount": 15990,
    "currency": "ARS"
  }
}
```

**Evento emitido (`order.paid.v1`)**
```json
{
  "eventId": "evt_002",
  "eventType": "order.paid.v1",
  "occurredAt": "2026-09-14T03:01:00Z",
  "tenantId": "tenant_001",
  "correlationId": "corr_abc",
  "payload": {
    "orderId": "ord_123",
    "paymentIntentId": "pi_789",
    "amount": 15990,
    "currency": "ARS"
  }
}
```

### Dependencias
- `ms-payments-service` (validación/firma y mapeo evento proveedor)
- `ms-shared-contracts` (envelope de eventos)
- Airtable (`payment_attempts`, `orders`, tabla de deduplicación)
- Broker/cola para publicar eventos

### Criterios de aceptación
- Webhook inválido por firma devuelve error y no persiste cambios.
- Eventos duplicados no vuelven a cambiar estado ni re-publicar.
- Al confirmar pago, orden pasa a `paid`.
- Se emite exactamente un `order.paid.v1` por pago confirmado.

### Definition of Done
- Webhook implementado con autenticidad + idempotencia.
- Tests de: firma inválida, evento duplicado, pago confirmado.
- Métricas/logs de resultado (`processed`, `duplicate`, `rejected`).

### Prompt para agente (copiar/pegar)
```text
Implementa `payments-async-fn` para procesar webhook de pago y emitir `order.paid.v1`.

Requisitos:
1) Endpoint POST /v1/payments/webhooks/provider.
2) Verificar firma del proveedor antes de procesar.
3) Deduplicar por `providerEventId` (idempotencia estricta).
4) Cuando el pago sea confirmado:
   - actualizar `payment_attempt` y `order` a estado pagado
   - emitir evento `order.paid.v1` con envelope estándar
5) Si es duplicado, responder OK sin efectos secundarios.
6) Contratos/eventos desde `ms-shared-contracts`.
7) Tests mínimos: firma inválida, duplicado, confirmed happy path.

No implementar notificaciones aquí; solo publicar el evento.
```

---

## Historia 5: `notifications-async-fn`

### Título
Implementar consumidor post-pago para envío de email con reintentos y DLQ.

### Objetivo
Consumir `order.paid.v1`, enviar email de confirmación y manejar fallos transitorios sin bloquear el flujo.

### Alcance
- Consumer de `order.paid.v1`.
- Construcción de payload de email.
- Envío vía `ms-notifications-service`.
- Reintentos con backoff.
- Envío a DLQ al agotar reintentos.

### Inputs / Outputs

**Input evento ejemplo**
```json
{
  "eventId": "evt_002",
  "eventType": "order.paid.v1",
  "tenantId": "tenant_001",
  "correlationId": "corr_abc",
  "payload": {
    "orderId": "ord_123",
    "customerEmail": "cliente@ejemplo.com",
    "amount": 15990,
    "currency": "ARS"
  }
}
```

**Output interno esperado**
- `ACK` si envío OK.
- `RETRY` si falla transitorio.
- `DLQ` si excede máximo de reintentos.

### Dependencias
- `ms-notifications-service`
- `ms-shared-contracts`
- Infra de cola + DLQ
- Airtable (`notifications` opcional para trazabilidad)

### Criterios de aceptación
- Procesa solo eventos `order.paid.v1` válidos.
- Es idempotente por `eventId`.
- Reintentos configurables (ej: 3).
- DLQ registra motivo de fallo definitivo.

### Definition of Done
- Consumer implementado con estrategia de reintentos.
- Tests: envío exitoso, error transitorio con retry, envío a DLQ.
- Observabilidad mínima por intento (`tenantId`, `eventId`, `attempt`).

### Prompt para agente (copiar/pegar)
```text
Implementa `notifications-async-fn` como consumidor de `order.paid.v1`.

Requisitos:
1) Consumir evento con envelope estándar y validar `tenantId`.
2) Llamar a `ms-notifications-service` para enviar email de confirmación.
3) Aplicar idempotencia por `eventId`.
4) Configurar reintentos con backoff para errores transitorios.
5) Si supera reintentos, enviar mensaje a DLQ con contexto de error.
6) Tests mínimos: success, retry, DLQ.
7) Logs estructurados y métricas básicas por intento.

Mantener implementación simple orientada a MVP.
```

---

## Historia 6: `ms-shared-contracts`

### Título
Implementar repositorio de contratos compartidos para requests/responses, eventos, enums y tipos comunes.

### Objetivo
Centralizar contratos versionados para evitar divergencias entre funciones sync, async y servicios.

### Alcance
- DTOs request/response v1 para commerce, coverage y payments.
- Envelope de eventos v1.
- Enums canónicos de estado (`order`, `payment`, `notification`).
- Reglas de versionado de contratos.

### Inputs / Outputs

**Envelope de evento v1 (ejemplo)**
```json
{
  "eventId": "evt_002",
  "eventType": "order.paid.v1",
  "occurredAt": "2026-09-14T03:01:00Z",
  "tenantId": "tenant_001",
  "correlationId": "corr_abc",
  "schemaVersion": "v1",
  "payload": {}
}
```

### Dependencias
- `docs/api-versioning-guidelines.md`
- `docs/architecture/event-driven-patterns.md`

### Criterios de aceptación
- Todos los contratos incluyen ejemplos válidos.
- Eventos comparten el mismo envelope.
- Cambios breaking exigen nueva versión.

### Definition of Done
- Paquete de contratos versionado y documentado.
- Validaciones automáticas de schema (si ya existe tooling).
- Changelog de contratos v1 inicial.

### Prompt para agente (copiar/pegar)
```text
Implementa `ms-shared-contracts` para centralizar contratos v1.

Requisitos:
1) Definir DTOs request/response para:
   - catálogo/carrito/orden
   - validación de cobertura
   - checkout
2) Definir envelope estándar de eventos y tipos para `order.paid.v1`.
3) Incluir enums compartidos de estados de orden/pago/notificación.
4) Incluir `tenantId` obligatorio en contratos relevantes.
5) Documentar reglas de compatibilidad y versionado.
6) Incluir ejemplos JSON válidos por contrato.

No incluir lógica de negocio ni dependencias de infraestructura.
```

---

## Historia 7: `ms-payments-service`

### Título
Implementar adapter desacoplado del proveedor de pagos.

### Objetivo
Encapsular integración con proveedor (crear intento, verificar firma, mapear estados) para que `payments-sync-fn` y `payments-async-fn` no dependan del SDK externo directamente.

### Alcance
- Método `createPaymentIntent`.
- Método `verifyWebhookSignature`.
- Método `mapProviderEventToDomain`.
- Normalización de errores del proveedor.

### Inputs / Outputs

**Input ejemplo `createPaymentIntent`**
```json
{
  "tenantId": "tenant_001",
  "orderId": "ord_123",
  "amount": 15990,
  "currency": "ARS",
  "idempotencyKey": "idem-pay-001"
}
```

**Output ejemplo**
```json
{
  "paymentIntentId": "pi_789",
  "checkoutUrl": "https://provider.example/checkout/abc",
  "providerStatus": "requires_action"
}
```

### Dependencias
- SDK/API del proveedor de pagos elegido
- `ms-shared-contracts`

### Criterios de aceptación
- API interna estable y agnóstica del proveedor.
- Errores del proveedor mapeados a errores de dominio.
- Soporta idempotencia enviada por caller.

### Definition of Done
- Adapter implementado con interfaz clara.
- Unit tests con mocks del proveedor.
- Documentación de configuración mínima por `tenantId`.

### Prompt para agente (copiar/pegar)
```text
Implementa `ms-payments-service` como adapter del proveedor de pagos.

Requisitos:
1) Exponer funciones internas:
   - createPaymentIntent
   - verifyWebhookSignature
   - mapProviderEventToDomain
2) Mantener interfaz agnóstica al proveedor (no filtrar modelo externo).
3) Propagar `tenantId` e idempotencia cuando aplique.
4) Estandarizar errores para consumo por sync/async functions.
5) Escribir unit tests con mocks del SDK/API.
6) Documentar variables de configuración requeridas.

No implementar endpoints HTTP aquí salvo que sean estrictamente necesarios para pruebas internas.
```

---

## Historia 8: `ms-notifications-service`

### Título
Implementar adapter de email y plantillas para notificaciones transaccionales.

### Objetivo
Proveer una capa reusable de envío de email desacoplada del proveedor para ser usada por `notifications-async-fn`.

### Alcance
- Método `sendOrderPaidEmail`.
- Render simple de plantilla MVP.
- Manejo de errores transitorios/permanentes.
- Soporte de configuración por `tenantId` (from/reply-to/template).

### Inputs / Outputs

**Input ejemplo**
```json
{
  "tenantId": "tenant_001",
  "to": "cliente@ejemplo.com",
  "template": "order-paid-v1",
  "data": {
    "orderId": "ord_123",
    "amount": 15990,
    "currency": "ARS"
  }
}
```

**Output ejemplo**
```json
{
  "messageId": "mail_456",
  "status": "sent"
}
```

### Dependencias
- Proveedor de email (SES/SendGrid/Mailgun u otro)
- `ms-shared-contracts`

### Criterios de aceptación
- API interna simple para enviar email transaccional.
- Distingue error transitorio de permanente.
- No expone secretos en logs.

### Definition of Done
- Adapter implementado con unit tests.
- Plantilla MVP funcional para orden pagada.
- Documentación de configuración por tenant.

### Prompt para agente (copiar/pegar)
```text
Implementa `ms-notifications-service` como adapter de email/plantillas.

Requisitos:
1) Exponer función sendOrderPaidEmail con input tipado.
2) Renderizar plantilla mínima `order-paid-v1`.
3) Soportar configuración por `tenantId` (from, reply-to, template vars).
4) Clasificar errores en transitorios vs permanentes para que el consumidor decida retry/DLQ.
5) No loguear secretos ni payload sensible completo.
6) Agregar unit tests del adapter y render de plantilla.

Mantener implementación MVP, desacoplada del proveedor.
```

---

## Matriz final sync vs async (resumen operativo)

- **Sync**: `commerce-sync-fn`, `coverage-sync-fn`, `payments-sync-fn`.
- **Async**: `payments-async-fn`, `notifications-async-fn`.
- **Reusable adapters**: `ms-payments-service`, `ms-notifications-service`.
- **Contratos compartidos**: `ms-shared-contracts`.

Con este backlog, cada agente puede tomar una historia y ejecutar un componente completo de MVP con límites claros, contratos explícitos y bajo riesgo de ambigüedad.
