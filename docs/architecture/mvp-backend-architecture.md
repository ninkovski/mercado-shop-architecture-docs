# MVP Backend Architecture

## Objetivo

Definir una arquitectura backend-first, desacoplada y de bajo costo para un MVP de e-commerce multi-tenant que soporte carrito, promociones, órdenes, checkout, pagos, webhooks, cobertura geográfica y notificaciones email.

## Principios

- Priorizar simplicidad operativa.
- Separar sincronía de asincronía.
- Mantener contratos versionados.
- Evitar sobreingeniería en el MVP.
- Diseñar desde el inicio para múltiples tiendas.

## Componentes principales

### 1. `commerce-sync-fn`
Responsable de:
- catálogo
- carrito
- promociones
- creación de órdenes
- consulta de estado de orden

### 2. `payments-sync-fn`
Responsable de:
- iniciar checkout
- crear intento de pago
- exponer estado de pago
- coordinar con proveedor de pagos

### 3. `payments-async-fn`
Responsable de:
- recibir webhooks de pago
- verificar firma
- procesar idempotencia
- confirmar o rechazar pagos
- emitir evento `payment.confirmed.v1`

### 4. `notifications-async-fn`
Responsable de:
- consumir eventos de orden/pago
- enviar email post-pago
- reintentos
- fallback a DLQ si falla el envío

### 5. `coverage-sync-fn`
Responsable de:
- validar si una dirección puede ser atendida
- calcular cobertura y costo base de envío
- devolver reglas aplicables antes del checkout

### 6. `ms-shared-contracts`
Responsable de:
- contratos de request/response
- eventos compartidos
- esquemas de validación
- enums canónicos

### 7. `ms-payments-service`
Responsable de:
- adapter del proveedor de pagos
- tokenización o creación de checkout session
- validación de estados remotos

### 8. `ms-notifications-service`
Responsable de:
- envío real de correo
- plantillas
- integración con proveedor

## Límites de contexto

### Commerce

Incluye:
- producto
- carrito
- promociones
- órdenes

Excluye:
- procesamiento interno de pagos
- envío de emails
- reglas de integración específicas de proveedor

### Payments

Incluye:
- intentos de pago
- checkout
- webhooks
- conciliación

Excluye:
- lógica de catálogo
- lógica de carrito
- contenido de marketing o email

### Notifications

Incluye:
- plantillas
- envío
- reintentos
- estado de entrega

Excluye:
- reglas de negocio de orden
- lógica de pago

### Coverage

Incluye:
- zonas permitidas
- validación geográfica
- reglas de costo

Excluye:
- checkout
- pagos
- notificaciones

## Flujo end-to-end del MVP

### Paso 1: carrito

El cliente agrega items al carrito.

```json
{
  "tenantId": "tenant_001",
  "cartId": "cart_123",
  "items": [
    {
      "productId": "prod_001",
      "quantity": 2
    }
  ]
}
```

### Paso 2: validación de cobertura

Antes de confirmar la orden se valida si la dirección puede ser atendida.

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

### Paso 3: creación de orden

`commerce-sync-fn` crea una orden con estado inicial `pending_payment`.

```json
{
  "orderId": "ord_123",
  "status": "pending_payment",
  "total": 15990,
  "currency": "ARS"
}
```

### Paso 4: checkout de pago

`payments-sync-fn` genera la sesión o intento de pago con el proveedor.

```json
{
  "orderId": "ord_123",
  "paymentIntentId": "pi_789",
  "checkoutUrl": "https://provider.example/checkout/abc"
}
```

### Paso 5: webhook de pago

El proveedor notifica el resultado de pago a `payments-async-fn`.

```json
{
  "eventType": "payment.confirmed.v1",
  "eventId": "evt_001",
  "paymentIntentId": "pi_789",
  "orderId": "ord_123",
  "status": "confirmed"
}
```

### Paso 6: confirmación de orden

Al confirmar el pago, la orden pasa a `paid` y se emite un evento para notificación.

```json
{
  "eventType": "order.paid.v1",
  "eventId": "evt_002",
  "orderId": "ord_123",
  "tenantId": "tenant_001"
}
```

### Paso 7: notificación email

`notifications-async-fn` envía el email post-pago.

## Interacciones síncronas

Las funciones síncronas deben:
- validar input
- cargar estado mínimo necesario
- ejecutar lógica de negocio corta
- persistir cambios pequeños
- responder rápido

No deben:
- esperar procesos largos
- llamar múltiples proveedores si eso incrementa latencia innecesaria
- contener lógica de integración pesada

## Interacciones asíncronas

Las funciones asíncronas deben:
- consumir eventos o webhooks
- verificar idempotencia
- intentar reenvío controlado
- publicar eventos derivados
- aislar fallos temporales

## Manejo de errores

### Errores funcionales

Ejemplos:
- dirección fuera de cobertura
- carrito inválido
- stock no disponible
- orden ya pagada
- pago rechazado

Respuesta ejemplo:

```json
{
  "error": {
    "code": "OUT_OF_COVERAGE",
    "message": "The delivery address is outside coverage.",
    "details": {
      "postalCode": "C1001"
    }
  }
}
```

### Errores técnicos

Ejemplos:
- timeout del proveedor
- error de red
- error de persistencia
- fallo en webhook parse

Reglas:
- registrar `correlationId`
- reintentar sólo si el error es transitorio
- evitar reintentos ciegos en errores de validación

## Seguridad mínima

1. Verificar firma de webhooks.
2. Validar origen de requests sensibles.
3. Exigir `Idempotency-Key` en checkout y creación de orden.
4. No exponer secretos en logs.
5. Segmentar acceso por `tenantId`.
6. Aplicar mínimo privilegio a servicios y tokens.
7. Sanitizar payloads entrantes.
8. Registrar auditoría de cambios críticos.

## Datos mínimos por request

### Campos comunes

- `tenantId`
- `correlationId`
- `idempotencyKey`
- `requestId`
- `actorType`

## Criterios de diseño para el MVP

### Debe existir desde el inicio

- API versionada
- separación sync/async
- contratos compartidos
- eventos con envelope común
- idempotencia
- observabilidad básica
- estructura multi-tenant

### No debe existir todavía

- microservicios excesivos
- orquestadores complejos
- colas múltiples por cada subcaso
- CQRS completo sin necesidad
- event sourcing total
- abstracciones sin uso real

## Estructura recomendada de responsabilidades

### Sync

- entrada de usuario
- validación
- comandos de negocio cortos
- lectura de estado

### Async

- webhooks
- notificaciones
- confirmaciones tardías
- retries
- procesamientos derivados

### Service

- adapters
- clients de proveedor
- lógica reusable
- mapeos de contratos externos

## Resultado esperado del MVP

El sistema debe permitir vender con este camino mínimo:

1. cliente agrega productos
2. valida cobertura
3. crea orden
4. inicia pago
5. confirma pago por webhook
6. notifica por email
7. deja trazabilidad para auditoría y soporte

## Regla final

Si una parte del sistema sólo existe para “organizar mejor” pero no ayuda a vender, validar o cobrar, debe diferirse.
