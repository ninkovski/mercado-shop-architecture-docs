# Event-Driven Patterns

## Objetivo

Definir patrones mínimos para eventos de dominio, webhooks, consumidores y procesos asíncronos con idempotencia, reintentos, DLQ y observabilidad.

## Cuándo usar eventos

Usar eventos cuando:
- una acción ya ocurrió y otros sistemas deben reaccionar
- el proceso puede ejecutarse de forma eventual
- el trabajo debe desacoplarse del request principal
- hay necesidad de reintentos o integración con terceros

No usar eventos cuando:
- la respuesta del usuario depende de completar todo en ese mismo request
- la operación es simple y directa
- el evento sólo reemplaza una llamada síncrona sin valor de desacoplamiento

## Event envelope común

Todos los eventos deben compartir una envoltura estándar.

```json
{
  "eventId": "evt_001",
  "eventType": "order.paid.v1",
  "eventVersion": "1",
  "tenantId": "tenant_001",
  "aggregateType": "order",
  "aggregateId": "ord_123",
  "correlationId": "corr_456",
  "idempotencyKey": "idem_789",
  "occurredAt": "2026-09-14T03:00:00Z",
  "data": {
    "orderId": "ord_123",
    "total": 15990,
    "currency": "ARS"
  }
}
```

## Eventos de dominio del MVP

### Commerce

- `cart.updated.v1`
- `order.created.v1`
- `order.paid.v1`
- `order.cancelled.v1`

### Payments

- `payment.initiated.v1`
- `payment.confirmed.v1`
- `payment.failed.v1`
- `payment.refunded.v1`

### Notifications

- `notification.email.requested.v1`
- `notification.email.sent.v1`
- `notification.email.failed.v1`

### Coverage

- `coverage.validated.v1`
- `coverage.rejected.v1`

## Idempotencia

### Regla general

Todo consumer, webhook handler y proceso de reintento debe ser idempotente.

### Estrategia mínima

- Persistir `eventId` o `idempotencyKey` procesado.
- Rechazar duplicados de forma segura.
- Asegurar que la segunda ejecución no duplique efectos.

### Ejemplo

Si un webhook `payment.confirmed.v1` llega dos veces:
- la primera vez confirma la orden
- la segunda vez debe responder OK sin repetir side effects

## Reintentos

### Reglas

- Reintentar sólo errores transitorios.
- No reintentar validaciones fallidas.
- Usar backoff exponencial con límite.
- Registrar cada intento con el mismo `correlationId`.

### Ejemplo de política

- intento 1: inmediato
- intento 2: +30s
- intento 3: +2m
- intento 4: +10m
- luego DLQ

## DLQ

### Cuándo enviar a DLQ

- fallo persistente del proveedor
- payload inválido no recuperable
- reintentos agotados
- dependencia externa fuera de servicio demasiado tiempo

### Reglas de DLQ

- cada mensaje en DLQ debe conservar su metadata completa
- debe ser inspeccionable manualmente
- debe existir proceso de replay controlado
- no debe perderse el contexto del error original

## Webhooks

### Reglas

- Verificar firma.
- Validar timestamp o ventana de aceptación si el proveedor lo permite.
- Nunca asumir orden de llegada.
- Tratar webhooks como delivery at-least-once.
- Confirmar recepción rápido y procesar internamente si el proveedor lo requiere.

### Flujo recomendado

1. recibir webhook
2. validar autenticidad
3. persistir evento crudo
4. deduplicar
5. procesar lógica de dominio
6. emitir evento derivado si corresponde

## Observabilidad

### Mínimos obligatorios

- `eventId`
- `correlationId`
- `tenantId`
- `aggregateId`
- `eventType`
- `attemptNumber`
- `handlerName`
- `status`

### Logs

Registrar:
- inicio de procesamiento
- decisión de idempotencia
- éxito
- error
- reintento
- envío a DLQ

### Métricas

Medir al menos:
- cantidad de eventos procesados
- cantidad de duplicados detectados
- cantidad de reintentos
- cantidad de fallos definitivos
- tiempo promedio de procesamiento

### Trazabilidad

Todo evento debe poder rastrearse desde:
- request original
- webhook inicial
- consumer
- evento derivado
- notificación final

## Contrato de consumidores

Los consumidores deben asumir:
- mensajes duplicados
- mensajes desordenados
- mensajes retrasados
- payloads extendidos con campos nuevos

No deben asumir:
- entrega exacta una sola vez
- orden perfecto entre eventos relacionados
- que el proveedor no duplicará mensajes

## Manejo de errores por tipo

### Error transitorio

Ejemplos:
- timeout
- red caída
- 5xx del proveedor

Acción:
- reintentar

### Error permanente

Ejemplos:
- firma inválida
- schema inválido
- entidad inexistente

Acción:
- no reintentar
- registrar y derivar a DLQ si corresponde

## Reglas de publicación de eventos

1. Publicar eventos sólo después de persistir el cambio de estado.
2. No publicar eventos fantasma.
3. Si no se puede garantizar publicación inmediata, usar outbox o mecanismo equivalente liviano.
4. No mezclar side effects externos y publicación de eventos sin control.

## Recomendación para MVP

Para empezar, usar:
- una envoltura de evento común
- deduplicación por `eventId`
- reintentos simples con backoff
- DLQ mínima
- logging estructurado

No empezar con:
- buses complejos innecesarios
- demasiados tipos de eventos
- contratos distintos por servicio sin necesidad

## Regla final

Si un proceso asíncrono no tiene idempotencia, observabilidad y estrategia de fallo, todavía no está listo para producción.
