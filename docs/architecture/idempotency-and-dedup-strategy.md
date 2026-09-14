# Estrategia de Idempotencia y Deduplicación (MVP)

## Objetivo
Evitar duplicados funcionales en APIs y consumidores async, garantizando resultados determinísticos.

---

## 1) Tabla de control transaccional (obligatoria)

## Nombre sugerido
`transaction_control`

## Columnas mínimas
- `id`
- `tenant_id`
- `operation` (ej: `CREATE_ORDER`, `CREATE_CHECKOUT`, `PROCESS_PROVIDER_WEBHOOK`, `SEND_ORDER_PAID_EMAIL`)
- `idempotency_key` (sync) o `source_event_id` (async)
- `request_hash` (hash estable del payload canónico)
- `status` (`PROCESSING`, `COMPLETED`, `FAILED`, `DUPLICATE`)
- `resource_id` (orderId, paymentIntentId, etc.)
- `response_snapshot` (json resumido)
- `error_code` (si aplica)
- `created_at`, `updated_at`
- `expires_at` (TTL lógico de la clave)

## Restricciones
- Unique index recomendado:
  - Sync: (`tenant_id`, `operation`, `idempotency_key`)
  - Async: (`tenant_id`, `operation`, `source_event_id`)

---

## 2) Idempotencia en flujos sync

## 2.1 `POST /v1/orders`
- Requiere `idempotencyKey`.
- Si la clave no existe: procesa y guarda resultado.
- Si existe con mismo `request_hash`: retorna mismo resultado funcional.
- Si existe con hash diferente: `IDEMPOTENCY_CONFLICT`.

## 2.2 `POST /v1/payments/checkout`
- Igual estrategia.
- Debe validar que la orden siga `pending_payment`.
- Reusar intento existente si es el mismo request funcional.

---

## 3) Deduplicación en flujos async

## 3.1 Webhook proveedor pagos
- Clave: `providerEventId`.
- Si evento ya procesado: responder 200/ACK sin side effects.
- Si nuevo: procesar transacción atómica:
  1) marcar control `PROCESSING`
  2) actualizar `payment_attempt` y `order`
  3) publicar `order.paid.v1`
  4) marcar `COMPLETED`

## 3.2 Consumer notificaciones
- Clave: `eventId`.
- Si duplicado: ACK inmediato sin reenvío de email.

---

## 4) Políticas de expiración

- Claves idempotentes sync: retención recomendada **30 días**.
- Dedupe webhook/eventos: retención recomendada **45 días**.
- Intentos de pago: expiran a **30 días** (`expiresAt` de negocio).

---

## 5) Concurrencia y consistencia

- Aplicar lock optimista/pesimista según datastore.
- Evitar publicar evento antes de persistir estado.
- Cada operación crítica debe ser atómica o compensable.

---

## 6) Observabilidad mínima

Log obligatorio por operación:
- `tenantId`
- `operation`
- `idempotencyKey` o `sourceEventId`
- `status`
- `resourceId`
- `correlationId`
