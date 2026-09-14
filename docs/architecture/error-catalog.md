# Catálogo de Errores (MVP)

## Objetivo
Unificar códigos de error para todos los servicios sync/async.

## Formato estándar
```json
{
  "code": "STRING_CODE",
  "message": "Human readable message",
  "details": [],
  "tenantId": "tenant_001",
  "correlationId": "corr_123"
}
```

---

## 1) Errores comunes

- `VALIDATION_ERROR` → request inválido
- `UNAUTHORIZED` → autenticación/autorización inválida
- `FORBIDDEN` → operación no permitida
- `RESOURCE_NOT_FOUND` → recurso inexistente
- `INTERNAL_ERROR` → error técnico no controlado
- `EXTERNAL_SERVICE_ERROR` → dependencia externa falló
- `TIMEOUT` → timeout de dependencia externa

---

## 2) Errores de dominio commerce

- `CART_NOT_FOUND`
- `CART_EMPTY`
- `INVALID_CART_ITEM_QUANTITY`
- `PRODUCT_NOT_FOUND`
- `ORDER_NOT_FOUND`
- `ORDER_INVALID_STATE`
- `PROMOTION_NOT_APPLICABLE`
- `PROMOTION_EXPIRED`

---

## 3) Errores de dominio coverage

- `COVERAGE_NOT_FOUND`
- `OUT_OF_COVERAGE`
- `INVALID_ADDRESS`

---

## 4) Errores de dominio payments

- `PAYMENT_ORDER_NOT_PENDING`
- `PAYMENT_ATTEMPT_NOT_FOUND`
- `PAYMENT_ALREADY_CONFIRMED`
- `PAYMENT_EXPIRED` (intento vencido > 30 días)
- `WEBHOOK_SIGNATURE_INVALID`
- `WEBHOOK_EVENT_UNSUPPORTED`

---

## 5) Errores de idempotencia/dedup

- `IDEMPOTENCY_KEY_REQUIRED`
- `IDEMPOTENCY_CONFLICT` (misma key, payload distinto)
- `DUPLICATE_EVENT_IGNORED` (informativo para async)

---

## 6) Errores de notificaciones

- `NOTIFICATION_INVALID_EVENT`
- `NOTIFICATION_PROVIDER_TEMPORARY_ERROR`
- `NOTIFICATION_PROVIDER_PERMANENT_ERROR`
- `NOTIFICATION_SENT_TO_DLQ`

---

## 7) Mapeo HTTP recomendado (sync)

- `VALIDATION_ERROR` → 400
- `UNAUTHORIZED` → 401
- `FORBIDDEN` → 403
- `RESOURCE_NOT_FOUND` → 404
- `ORDER_INVALID_STATE` / `PAYMENT_ORDER_NOT_PENDING` → 409
- `IDEMPOTENCY_CONFLICT` → 409
- `TIMEOUT` / `EXTERNAL_SERVICE_ERROR` → 502/504
- `INTERNAL_ERROR` → 500
