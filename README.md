# Mercado Shop Architecture Docs

Documentación de arquitectura, estándares y decisiones base para una plataforma de e-commerce backend-first, reusable para múltiples tiendas virtuales.

## Visión de plataforma

Mercado Shop es una plataforma modular para múltiples tiendas virtuales, diseñada para iniciar con un MVP de bajo costo operativo y crecer sin acoplarse a un rubro específico. El foco inicial es el backend: funciones serverless, APIs versionadas, eventos asíncronos, integraciones desacopladas y un modelo de datos ligero con Airtable.

La plataforma debe permitir que cada tienda comparta capacidades comunes, pero mantenga configuraciones, catálogos, promociones, reglas de cobertura y flujos de pago propios.

## Principios de diseño

1. **Backend-first**: el sistema se diseña desde capacidades de negocio expuestas por APIs.
2. **Bajo costo operativo**: preferencia por Functions, colas administradas y servicios ligeros.
3. **Desacoplamiento**: pagos, notificaciones y procesos async deben ser reemplazables.
4. **Versionado explícito**: toda API pública vive bajo versión (`/v1`, `/v2`, etc.).
5. **Idempotencia por defecto**: webhooks, reintentos y consumers deben tolerar duplicados.
6. **MVP antes que plataforma completa**: primero el flujo mínimo que vende, luego la optimización.
7. **Multi-tenant by design**: todo debe contemplar `tenant/store` desde el inicio.
8. **Contrato primero**: documentación y esquemas antes de nuevas implementaciones.

## Qué incluye esta documentación

- Convenciones de nombres de repositorios
- Reglas de versionado de APIs
- Arquitectura MVP backend
- Patrones event-driven y async
- Esquema mínimo de Airtable
- Backlog de tareas listas para agentes
- Starter pack de repos y skills por componente

## Mapa de repos futuros

La documentación se usará para crear repos separados por capacidad.

### Repos de documentación y contratos

- `mercado-shop-architecture-docs`: documentación y estándares
- `ms-shared-contracts`: DTOs, eventos, esquemas y contratos compartidos

### Repos de funciones síncronas (`request/response`)

- `ms-commerce-sync-fn`: catálogo, carrito, promociones, órdenes
- `ms-payments-sync-fn`: checkout, creación de intentos de pago, confirmación síncrona
- `ms-coverage-sync-fn`: validación de cobertura geográfica y costo de envío

### Repos de funciones asíncronas (`event/webhook/consumer`)

- `ms-payments-async-fn`: webhooks de pago, conciliación, eventos de pago
- `ms-notifications-async-fn`: emails, reintentos, plantillas, colas
- `ms-commerce-async-fn`: proyección de eventos de órdenes, actualizaciones derivadas
- `ms-coverage-async-fn`: sincronización de zonas, recalculo de reglas si aplica

### Repos de servicios reusables

- `ms-payments-service`: integración con proveedor de pagos
- `ms-notifications-service`: envío de email/SMS/push
- `ms-coverage-service`: reglas geográficas/cálculo de entrega
- `ms-commerce-service`: lógica reusable de dominio si el tamaño lo justifica

### Frontend futuro

- `mercado-shop-web`: frontend estático / SPA / storefront

## Dominios iniciales

- **commerce**: catálogo, carrito, promociones, órdenes
- **payments**: checkout, confirmación, conciliación
- **notifications**: email, plantillas, reintentos
- **coverage**: validación geográfica y costo de envío

## Flujo de alto nivel

1. El cliente agrega productos al carrito.
2. El backend valida disponibilidad, promociones y cobertura.
3. Se crea la orden con estado `pending_payment`.
4. El servicio de pagos genera el intento de pago.
5. El proveedor confirma pago vía webhook.
6. El backend confirma la orden.
7. Se dispara un evento asíncrono para notificar por email.

## Decisiones arquitectónicas base

- Airtable se usa como base ligera para MVP, no como base definitiva de escala.
- Las integraciones externas se encapsulan en servicios o adapters.
- Las funciones síncronas sólo coordinan casos de uso; la lógica pesada vive en servicios compartidos.
- Los procesos largos, reintentos y webhooks van a funciones asíncronas.
- Todo evento importante debe tener `event_id`, `tenant_id`, `aggregate_id`, `correlation_id` e `idempotency_key`.

## Qué va en sync vs async

### Sync

Usar sincronía cuando el usuario necesita respuesta inmediata:

- consultar catálogo
- gestionar carrito
- validar cobertura
- crear orden
- iniciar checkout
- obtener estado de pago/orden

### Async

Usar asincronía cuando el proceso puede desacoplarse o reintentarse:

- webhooks de pago
- confirmación eventual de pago
- envío de emails
- proyecciones derivadas
- conciliación
- notificaciones y reintentos

## Objetivo de esta base documental

Antes de crear repos de implementación, esta documentación define:

- nombres consistentes
- fronteras de contexto
- versión de APIs
- estructura de eventos
- esquema mínimo de datos
- tareas listas para agentes

## Starter packs para implementación

- `docs/backlog/repo-starter-packs.md`: repos acordados + starter pack de skills por componente.

## Próximos pasos recomendados

1. Aprobar convenciones de nombres.
2. Aprobar reglas de versionado de API.
3. Validar arquitectura MVP.
4. Definir contratos y esquemas base.
5. Crear repos de implementación por dominio.
