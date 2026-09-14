# Convenciones de naming para repositorios

Estas convenciones existen para que cualquier persona o agente identifique rápido si un repo contiene sincronía, asincronía, servicios compartidos o contratos.

## Regla general

Usar prefijo `ms-` para cualquier repo de implementación relacionado con Mercado Shop.

## Formato base

- `ms-<dominio>-sync-fn`
- `ms-<dominio>-async-fn`
- `ms-<capability>-service`
- `ms-shared-contracts`
- `mercado-shop-architecture-docs`
- `mercado-shop-web`

## Significado de cada sufijo

### `-sync-fn`

Repositorio de Functions síncronas tipo request/response.

Uso típico:
- endpoints HTTP
- validaciones inmediatas
- comandos del usuario
- lectura/escritura corta

Ejemplos:
- `ms-commerce-sync-fn`
- `ms-payments-sync-fn`
- `ms-coverage-sync-fn`

### `-async-fn`

Repositorio de Functions asíncronas.

Uso típico:
- consumers de cola
- procesamiento de webhooks
- reintentos
- tareas derivadas
- eventos de dominio

Ejemplos:
- `ms-payments-async-fn`
- `ms-notifications-async-fn`
- `ms-commerce-async-fn`

### `-service`

Repositorio de servicios integradores o reutilizables.

Uso típico:
- cliente de proveedor externo
- lógica reusable de integración
- adaptadores
- SDK interno ligero

Ejemplos:
- `ms-payments-service`
- `ms-notifications-service`
- `ms-coverage-service`

### `-contracts`

Repositorio para contratos compartidos.

Uso típico:
- eventos de dominio
- DTOs
- schemas JSON
- tipos compartidos
- validaciones de request/response

Ejemplo:
- `ms-shared-contracts`

### `-docs`

Repositorio de documentación y estándares.

Ejemplo:
- `mercado-shop-architecture-docs`

## Convención por dominio

Usar dominios estables y de negocio, no nombres de tecnología.

Dominios iniciales recomendados:
- `commerce`
- `payments`
- `notifications`
- `coverage`

### Ejemplo correcto

- `ms-commerce-sync-fn`
- `ms-payments-async-fn`
- `ms-notifications-service`
- `ms-shared-contracts`

### Ejemplo incorrecto

- `ms-node-api`
- `ms-lambda-payments`
- `shop-functions-v2`
- `backend-utils`

## Reglas estrictas

1. **No incluir tecnología en el nombre** salvo cuando aporte claridad real y sea parte del rol del repo.
2. **No mezclar sync y async en un mismo repo** si eso complica despliegue, ownership o escalado.
3. **No usar nombres genéricos** como `api`, `backend`, `common`.
4. **No usar abreviaciones ambiguas**.
5. **Un repo = una responsabilidad principal**.
6. **El nombre debe indicar el tipo de ejecución** si es una función.
7. **Los contratos deben vivir separados** de la implementación cuando sean compartidos por más de un repo.

## Criterio para decidir sync vs async

### Elegir `sync-fn` si:
- el usuario espera respuesta inmediata
- el caso de uso es corto
- hay validación previa al commit de negocio
- el endpoint forma parte de una interacción directa del storefront

### Elegir `async-fn` si:
- el trabajo puede terminar después
- hay webhooks o colas
- hay reintentos
- la operación puede duplicarse y debe ser idempotente
- el costo de espera es mayor que el valor de la respuesta inmediata

## Criterio para decidir `service`

Elegir `service` cuando el repo encapsule una capacidad reusable y no sea el punto de entrada HTTP principal.

Ejemplos:
- SDK de pasarela de pago
- adaptador de email
- cliente para Airtable
- motor de reglas reutilizable

## Repos compartidos

### `ms-shared-contracts`

Debe contener únicamente:
- esquemas compartidos
- contratos de eventos
- modelos canónicos
- utilidades de validación livianas

No debe contener:
- lógica de negocio de dominio
- acceso a infraestructura
- funciones deployment-specific

## Versionado de nombres

Evitar poner versiones en el nombre del repo salvo necesidad extrema.

Correcto:
- `ms-payments-sync-fn`

No recomendado:
- `ms-payments-sync-fn-v1`

La versión debe vivir en la API y en los contratos, no en el nombre del repo.

## Convención de carpetas dentro de repos de implementación

Sugerencia mínima común:

- `src/`
- `src/functions/`
- `src/services/`
- `src/domain/`
- `src/contracts/`
- `src/infrastructure/`
- `docs/`
- `tests/`

## Regla operativa para el equipo

Si un nuevo repo no puede describirse en una sola frase usando uno de estos patrones:
- `sync fn`
- `async fn`
- `service`
- `contracts`
- `docs`

entonces el repo todavía no está bien acotado.
