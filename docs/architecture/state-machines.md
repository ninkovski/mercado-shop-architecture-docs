# State Machines (MVP)

## Objetivo
Definir transiciones de estado válidas para evitar inconsistencias entre servicios sync/async.

---

## 1) Orden (`order`)

### Estados
- `draft`
- `pending_payment`
- `paid`
- `payment_expired`
- `cancelled` (reservado, fuera de MVP operativo)

### Transiciones válidas
- `draft -> pending_payment` (al crear orden desde carrito válido)
- `pending_payment -> paid` (al confirmar pago)
- `pending_payment -> payment_expired` (al vencer ventana de pago de 30 días)
- `draft -> cancelled` (opcional futuro)
- `pending_payment -> cancelled` (opcional futuro, reglas negocio)

### Reglas
- No se permite `paid -> pending_payment`.
- No se permite crear checkout para `paid` o `payment_expired`.
- Toda orden en `pending_payment` debe tener snapshot de precios/promos.

---

## 2) Intento de pago (`payment_attempt`)

### Estados
- `created`
- `requires_action`
- `processing`
- `succeeded`
- `failed`
- `expired`

### Transiciones válidas
- `created -> requires_action` (checkout URL/token generado)
- `requires_action -> processing` (proveedor confirma recepción/flujo)
- `processing -> succeeded` (webhook confirmado)
- `processing -> failed` (rechazo definitivo)
- `requires_action -> expired` (vencimiento 30 días)
- `created -> expired` (sin completar flujo)

### Reglas
- `payment_attempt.expiresAt = createdAt + 30 días`.
- Si un intento vence, no puede mutar a `succeeded` sin nuevo intento.
- Idempotencia por `idempotencyKey` en creación de checkout.

---

## 3) Notificación (`notification`)

### Estados
- `pending`
- `sent`
- `retrying`
- `dead_letter`

### Transiciones válidas
- `pending -> sent` (envío exitoso)
- `pending -> retrying` (error transitorio)
- `retrying -> sent` (reintento exitoso)
- `retrying -> dead_letter` (agota retries)

### Reglas
- Idempotencia por `eventId`.
- Máximo retries configurable (default MVP: 3).
- Cada intento debe registrar `attempt`, `reason`, `timestamp`.

---

## 4) Promociones (`promotion`)

### Estados lógicos
- `active`
- `expired`
- `disabled` (manual)

### Reglas de vigencia
- Vigencia estándar MVP: **2 días** desde `startAt` (o hasta `endAt` explícito).
- Al crear orden: se evalúa promo vigente y se guarda **snapshot**.
- Si promo expira luego, **NO** cambia el total de orden ya creada.
