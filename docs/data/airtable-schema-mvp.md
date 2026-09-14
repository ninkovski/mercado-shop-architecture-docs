# Airtable Schema MVP

## Objetivo

Definir el modelo mínimo de Airtable para soportar el MVP de Mercado Shop con bajo costo operativo y suficiente flexibilidad para múltiples tiendas.

## Principios del esquema

- Mantener pocas tablas.
- Separar configuración por `tenant`.
- Evitar normalización excesiva.
- Guardar relaciones por IDs estables.
- Permitir lectura simple desde Functions.
- No intentar modelar todo el dominio en la primera versión.

## Tablas mínimas

### 1. `tenants`

Representa cada tienda virtual.

Campos:
- `tenant_id` (string, único)
- `name` (string)
- `slug` (string, único)
- `status` (single select: active, inactive)
- `currency` (string)
- `country` (string)
- `created_at` (datetime)

Relaciones:
- uno a muchos con productos, promociones, órdenes, cobertura y configuración

### 2. `products`

Catálogo del tenant.

Campos:
- `product_id` (string, único)
- `tenant_id` (link a tenants)
- `sku` (string)
- `name` (string)
- `description` (long text)
- `price` (number)
- `currency` (string)
- `active` (checkbox)
- `stock_qty` (number)
- `category` (string)
- `created_at` (datetime)
- `updated_at` (datetime)

### 3. `carts`

Carritos activos o abandonados.

Campos:
- `cart_id` (string, único)
- `tenant_id` (link a tenants)
- `customer_id` (string, nullable)
- `status` (single select: active, converted, abandoned)
- `currency` (string)
- `subtotal` (number)
- `discount_total` (number)
- `shipping_total` (number)
- `total` (number)
- `created_at` (datetime)
- `updated_at` (datetime)

### 4. `cart_items`

Items del carrito.

Campos:
- `cart_item_id` (string, unique)
- `cart_id` (link a carts)
- `product_id` (link a products)
- `quantity` (number)
- `unit_price` (number)
- `line_total` (number)
- `created_at` (datetime)

### 5. `promotions`

Promociones y ofertas activas.

Campos:
- `promotion_id` (string, único)
- `tenant_id` (link a tenants)
- `name` (string)
- `type` (single select: percentage, fixed_amount, free_shipping)
- `value` (number)
- `start_at` (datetime)
- `end_at` (datetime)
- `active` (checkbox)
- `conditions_json` (long text JSON)
- `created_at` (datetime)

### 6. `orders`

Órdenes creadas por checkout.

Campos:
- `order_id` (string, único)
- `tenant_id` (link a tenants)
- `cart_id` (link a carts)
- `customer_id` (string)
- `status` (single select: pending_payment, paid, cancelled, failed, refunded)
- `subtotal` (number)
- `discount_total` (number)
- `shipping_total` (number)
- `total` (number)
- `currency` (string)
- `payment_status` (string)
- `coverage_status` (single select: pending, valid, invalid)
- `created_at` (datetime)
- `updated_at` (datetime)

### 7. `order_items`

Snapshot de items de la orden.

Campos:
- `order_item_id` (string, unique)
- `order_id` (link a orders)
- `product_id` (string)
- `product_name_snapshot` (string)
- `unit_price_snapshot` (number)
- `quantity` (number)
- `line_total` (number)

### 8. `payment_attempts`

Intentos de pago y su trazabilidad.

Campos:
- `payment_attempt_id` (string, único)
- `tenant_id` (link a tenants)
- `order_id` (link a orders)
- `provider` (string)
- `provider_payment_id` (string)
- `status` (single select: initiated, confirmed, failed, refunded)
- `amount` (number)
- `currency` (string)
- `checkout_url` (url)
- `webhook_payload_json` (long text JSON)
- `created_at` (datetime)
- `updated_at` (datetime)

### 9. `coverage_zones`

Zonas habilitadas por tenant.

Campos:
- `coverage_zone_id` (string, único)
- `tenant_id` (link a tenants)
- `country` (string)
- `state` (string)
- `city` (string)
- `postal_code` (string)
- `shipping_cost` (number)
- `active` (checkbox)
- `created_at` (datetime)

### 10. `notifications`

Registro de notificaciones enviadas.

Campos:
- `notification_id` (string, único)
- `tenant_id` (link a tenants)
- `order_id` (link a orders)
- `type` (single select: email, sms, push)
- `template_key` (string)
- `recipient` (string)
- `status` (single select: queued, sent, failed, retried)
- `provider_message_id` (string)
- `last_error` (long text)
- `created_at` (datetime)
- `updated_at` (datetime)

## Relaciones principales

- `tenants` 1:N `products`
- `tenants` 1:N `promotions`
- `tenants` 1:N `carts`
- `carts` 1:N `cart_items`
- `carts` 1:1 `orders` (en práctica puede ser 1:N histórico, pero el MVP usa una conversión principal)
- `orders` 1:N `order_items`
- `orders` 1:N `payment_attempts`
- `tenants` 1:N `coverage_zones`
- `orders` 1:N `notifications`

## Campos obligatorios transversales

Todo registro relevante debe tener al menos:
- `tenant_id`
- `created_at`
- `updated_at` cuando aplique
- un identificador estable y único

## Qué no modelar todavía

- devoluciones complejas
- múltiples almacenes
- split shipments
- loyalty points
- pricing engine avanzado
- catálogos multi-idioma
- reglas dinámicas demasiado genéricas

## Reglas de uso desde Functions

- Leer por IDs estables.
- No hacer scans completos sin filtro.
- Mantener `tenant_id` como primer filtro lógico.
- Escribir snapshots en `orders` y `order_items`.
- Guardar el payload crudo del webhook cuando sea útil para auditoría.

## Regla final

Si un nuevo campo no ayuda a vender, cobrar, validar cobertura o notificar, no debe entrar en el MVP.
