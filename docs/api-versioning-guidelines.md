# API Versioning Guidelines

## Objetivo

Establecer reglas estrictas para versionar APIs públicas de Mercado Shop sin romper integraciones existentes y sin generar deuda innecesaria.

## Principios

1. Toda API pública debe tener versión explícita en la URL.
2. La primera versión estable será `/v1`.
3. Los cambios incompatibles requieren nueva versión mayor.
4. La compatibilidad hacia atrás se mantiene mientras sea razonable.
5. La deprecación debe ser comunicada y fechada.
6. Las APIs internas pueden evolucionar distinto, pero los contratos públicos no deben improvisarse.

## Convención de versión en URL

### Reglas

- Usar prefijo de ruta: `/v1`, `/v2`, etc.
- No usar versiones en query params.
- No usar versiones en headers como mecanismo principal de routing.
- No exponer endpoints sin versión si son públicos.

### Ejemplos

Correcto:

```http
GET /v1/products
POST /v1/carts/{cartId}/items
POST /v1/orders
POST /v1/payments/checkout
```

Incorrecto:

```http
GET /products?version=1
GET /api/products
```

## Qué se considera breaking change

Cualquier cambio que pueda romper clientes existentes debe tratarse como breaking.

### Ejemplos de breaking changes

- Renombrar campos JSON.
- Eliminar campos existentes.
- Cambiar tipo de dato.
- Cambiar el significado semántico de un campo.
- Cambiar códigos de estado esperados.
- Cambiar la forma de autenticación requerida.
- Alterar el contrato de webhook.
- Cambiar el orden o estructura de listas cuando el consumidor depende de ello.
- Modificar el idempotency model.

### Ejemplos no breaking

- Agregar campos nuevos opcionales.
- Agregar nuevos endpoints.
- Agregar headers opcionales.
- Agregar nuevos valores a enums de forma compatible.
- Mejorar mensajes de error sin cambiar el código base.

## Reglas para compatibilidad hacia atrás

1. Nunca remover un campo sin pasar por ciclo formal de deprecación.
2. Nunca cambiar el tipo de un campo en la misma versión.
3. Todo campo nuevo debe ser opcional por defecto si el cliente actual puede ignorarlo.
4. Los consumidores deben tolerar campos desconocidos.
5. Los productores no deben asumir que el consumidor lee todos los campos.

## Estrategia de evolución

### Extensiones compatibles

Se permiten dentro de la misma versión:
- agregar `statusDetail`
- agregar `metadata`
- agregar `shippingCostBreakdown`
- agregar `providerReference`

### Cambios mayores

Requieren nueva versión cuando:
- se cambia el modelo de orden
- se renombra `paymentStatus` por `status`
- se reestructura un webhook
- se cambia el contrato de creación de carrito

## Política de deprecación

### Reglas

- Anunciar deprecación antes de remover.
- Mantener la versión anterior durante una ventana de transición.
- Documentar fecha de inicio y fecha límite.
- Publicar migración sugerida.

### Recomendación mínima

- Aviso previo: 60-90 días si hay clientes externos.
- Mantener compatibilidad durante la ventana acordada.
- Registrar deprecación en la documentación y en el changelog.

## Headers recomendados

### Respuesta

Usar headers para comunicar estado de la API:

- `Deprecation: true`
- `Sunset: <RFC 1123 date>`
- `Link: <migration-doc-url>; rel="deprecation"`

Ejemplo:

```http
Deprecation: true
Sunset: Tue, 30 Dec 2026 23:59:59 GMT
Link: <https://github.com/ninkovski/mercado-shop-architecture-docs/blob/main/docs/api-versioning-guidelines.md>; rel="deprecation"
```

### Request

Recomendados para observabilidad e idempotencia:

- `Idempotency-Key`: necesario en comandos sensibles como checkout y creación de orden.
- `X-Correlation-Id`: rastreo entre servicios y eventos.
- `X-Tenant-Id`: sólo si no viene embebido en token o contexto de autenticación.

## Reglas por tipo de endpoint

### Endpoints de lectura

- Pueden evolucionar con nuevos campos opcionales.
- Deben mantenerse estables en nombres y tipos.

### Endpoints de escritura

- Requieren validación estricta de contrato.
- Deben ser idempotentes cuando el caso de uso lo permita.
- Deben devolver identificadores estables.

### Webhooks

- Deben versionarse explícitamente.
- Deben incluir `event_type`, `event_version` y `event_id`.
- No deben cambiar sin migración formal.

## Reglas para errores

Usar estructura consistente por versión.

Ejemplo:

```json
{
  "error": {
    "code": "ORDER_ALREADY_PAID",
    "message": "The order has already been paid.",
    "details": {
      "orderId": "ord_123"
    }
  }
}
```

Reglas:
- `code` estable y semántico.
- `message` legible para humanos.
- `details` opcional.
- No depender del texto de `message` para lógica de cliente.

## Reglas de compatibilidad para eventos

Los eventos también son contratos versionados.

Ejemplo de naming:
- `order.created.v1`
- `payment.confirmed.v1`
- `notification.email.requested.v1`

Cambios incompatibles en un evento requieren `v2`.

## Checklist antes de publicar una versión

- ¿Rompe clientes existentes?
- ¿Se documentó el cambio?
- ¿Existe plan de migración?
- ¿Se actualizó contrato y ejemplos?
- ¿Se revisó idempotencia?
- ¿Se actualizó observabilidad?

## Regla final

Si el cambio obliga a un consumidor a reescribir su integración, entonces no debe ir en la misma versión.
