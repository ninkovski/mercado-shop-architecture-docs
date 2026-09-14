# MVP Runbooks Operativos

## Objetivo
Definir respuesta operativa mínima ante incidentes comunes del flujo order-payment-notification.

---

## 1) Incidente: webhook de pagos falla masivamente

## Señales
- aumento `WEBHOOK_SIGNATURE_INVALID`
- caída de `order.paid.v1` emitidos
- backlog en cola de eventos de pago

## Acciones
1. Verificar secreto/firma configurada por tenant.
2. Confirmar hora del servidor y tolerancia de timestamp.
3. Revisar cambios recientes en `ms-payments-service`.
4. Habilitar alerta de tasa de rechazo > umbral.
5. Reprocesar eventos válidos desde origen/cola si aplica.

---

## 2) Incidente: crecimiento de DLQ en notificaciones

## Señales
- aumento de `events_dlq_total`
- errores permanentes del proveedor email

## Acciones
1. Clasificar causa (`temporary` vs `permanent`).
2. Revisar configuración tenant (`from`, `reply-to`, template).
3. Si es temporal, ajustar retry/backoff.
4. Si es permanente, corregir payload/template y re-drive de DLQ.
5. Documentar impacto por tenant.

---

## 3) Incidente: duplicados de pago detectados

## Señales
- múltiples webhooks mismo `providerEventId`
- más de un `payment_attempt` equivalente

## Acciones
1. Validar unique constraints en `transaction_control`.
2. Verificar dedup antes de side effects.
3. Auditar uso de `idempotencyKey` en checkout.
4. Ejecutar script de reconciliación sobre órdenes afectadas.
5. Activar métrica/alarma de `duplicates_rate`.

---

## 4) Incidente: timeouts en coverage o payments sync

## Señales
- aumento p95/p99
- `TIMEOUT` y `EXTERNAL_SERVICE_ERROR`

## Acciones
1. Confirmar latencia de dependencia externa.
2. Aplicar circuit breaker simple (si ya está habilitado).
3. Reducir timeout a valores controlados + retry acotado.
4. Degradar funcionalmente donde aplique (sin romper contrato).
5. Escalar capacidad solo si saturación real de CPU/memoria.

---

## 5) Operación diaria (checklist)

- [ ] Salud endpoints `/health` y `/ready`.
- [ ] Errores 5xx por servicio dentro de umbral.
- [ ] DLQ con volumen normal.
- [ ] Retries dentro de tasa esperada.
- [ ] Sin crecimiento anómalo de `IDEMPOTENCY_CONFLICT`.
- [ ] Verificación de expiración:
  - payment attempts > 30 días pasan a `expired`
  - promociones > 2 días no aplican a nuevas órdenes

---

## 6) Reprocesamiento seguro

## Regla general
Siempre re-procesar con idempotencia activa.

## Pasos
1. Identificar lote (tenant, rango horario, tipo evento).
2. Simular impacto (dry-run si tooling disponible).
3. Reprocesar en lotes pequeños.
4. Verificar métricas y consistencia de estados.
5. Documentar postmortem breve.
