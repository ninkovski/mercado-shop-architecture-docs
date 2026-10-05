# Repos MVP + starter pack de skills

Documento operativo para crear los repos acordados en historias de usuario y arrancar cada componente con un set mínimo de skills.

## Repos a crear (según historias)

1. `ms-commerce-sync-fn`
2. `ms-coverage-sync-fn`
3. `ms-payments-sync-fn`
4. `ms-payments-async-fn`
5. `ms-notifications-async-fn`
6. `ms-shared-contracts`
7. `ms-payments-service`
8. `ms-notifications-service`

> Fuente: `docs/backlog/implementation-stories.md` + `docs/repo-naming-conventions.md`.

## Starter pack común (aplica a todos)

- **Contract-first**: modelar requests/responses/eventos desde `ms-shared-contracts`.
- **Arquitectura hexagonal**: `application`, `domain`, `infrastructure`, `shared`.
- **Observabilidad mínima**: logs estructurados con `tenantId` y `correlationId`.
- **Testing mínimo**: happy path + validaciones + errores clave.
- **Seguridad base**: no secretos en logs, validaciones de input y manejo homogéneo de errores.

## Starter pack por repo

### 1) `ms-commerce-sync-fn`
- Skills foco: diseño de APIs síncronas `/v1`, idempotencia en creación de orden, snapshot de precios/promos.
- Entradas clave: catálogo, carrito, orden, cobertura.
- Dependencias: `ms-shared-contracts`, `ms-coverage-sync-fn`.

### 2) `ms-coverage-sync-fn`
- Skills foco: matching geográfico determinístico, cálculo de shipping, validación de request.
- Entradas clave: país/estado/ciudad/código postal por `tenantId`.
- Dependencias: `ms-shared-contracts`.

### 3) `ms-payments-sync-fn`
- Skills foco: checkout síncrono, idempotencia por `idempotencyKey`, validación de estado de orden.
- Entradas clave: orden `pending_payment`, monto y moneda.
- Dependencias: `ms-shared-contracts`, `ms-payments-service`.

### 4) `ms-payments-async-fn`
- Skills foco: webhooks seguros, verificación de firma, deduplicación estricta, emisión de evento de dominio.
- Entradas clave: `providerEventId`, `paymentIntentId`, estado de pago.
- Dependencias: `ms-shared-contracts`, `ms-payments-service`.

### 5) `ms-notifications-async-fn`
- Skills foco: consumo de eventos, retries con backoff, DLQ, idempotencia por `eventId`.
- Entradas clave: `order.paid.v1`.
- Dependencias: `ms-shared-contracts`, `ms-notifications-service`.

### 6) `ms-shared-contracts`
- Skills foco: diseño de DTOs y eventos versionados, compatibilidad backward, enums canónicos.
- Entradas clave: contratos v1 de commerce, coverage, payments y eventos.
- Dependencias: guías de versionado y eventos.

### 7) `ms-payments-service`
- Skills foco: adapter de proveedor de pagos, normalización de errores, interfaz agnóstica.
- Entradas clave: create intent, verify signature, map provider event.
- Dependencias: SDK/API del proveedor + `ms-shared-contracts`.

### 8) `ms-notifications-service`
- Skills foco: adapter de email, render de template transaccional, clasificación de errores transitorio/permanente.
- Entradas clave: `sendOrderPaidEmail` con configuración por tenant.
- Dependencias: proveedor de email + `ms-shared-contracts`.

## Prompt base (starter) para cualquier repo

```text
Implementa el repo <REPO_NAME> siguiendo estrictamente:
1) docs/backlog/implementation-stories.md
2) docs/backlog/development-standards-and-stack-rationale.md

Obligatorio:
- Java 21 + Quarkus 3.x
- Arquitectura hexagonal
- Contratos desde ms-shared-contracts
- tenantId y correlationId en todos los flujos
- Manejo de errores estándar
- Idempotencia donde aplique
- Tests mínimos (happy path + validaciones + errores clave)
- Sin sobreingeniería fuera de alcance MVP
```
