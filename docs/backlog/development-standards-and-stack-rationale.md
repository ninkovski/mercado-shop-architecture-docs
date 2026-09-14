# Estándares de desarrollo + justificación de stack (agent-ready)

## 1) Decisión de stack (oficial)

**Stack oficial MVP:** **Java 21 + Quarkus 3.x**

### 1.1 Justificación (contexto: recursos limitados)
Elegimos Quarkus sobre alternativas Java más pesadas para el MVP porque:

1. **Eficiencia de recursos (RAM/CPU):** menor huella típica en servicios pequeños.
2. **Menor costo operativo:** permite correr más componentes en instancias pequeñas.
3. **Arranque rápido:** útil para cargas variables y despliegues frecuentes.
4. **Productividad Java:** mantenemos tipado fuerte, ecosistema maduro, tooling robusto.
5. **Escalabilidad progresiva:** permite evolucionar sin rehacer lenguaje ni contratos.

### 1.2 Criterios de arquitectura derivados
- Priorizar simplicidad operativa sobre complejidad distribuida.
- Evitar sobrefragmentación de microservicios en etapa MVP.
- Definir contratos primero (`ms-shared-contracts`) y luego implementación.
- Idempotencia y trazabilidad como requisitos no negociables.

---

## 2) Lineamientos obligatorios para TODO desarrollo

### 2.1 Principios transversales
- `tenantId` obligatorio en requests/responses/eventos.
- `correlationId` obligatorio para trazabilidad.
- Contratos HTTP/eventos se importan desde `ms-shared-contracts`.
- Manejo de errores homogéneo (`code`, `message`, `details`).
- No implementar capacidades fuera del alcance de cada historia.

### 2.2 Estructura de código estándar
```text
src/main/java/com/mercadoshop/<service>/
  application/      # casos de uso y puertos
  domain/           # reglas y modelos de negocio
  infrastructure/   # adapters (HTTP, DB, broker, proveedores)
  shared/           # utilidades transversales (logging, idempotencia, mappers)
```

### 2.3 Patrones permitidos/recomendados
- **Hexagonal (Ports & Adapters)** como estructura principal.
- **Adapter Pattern** para proveedores externos (`ms-payments-service`, `ms-notifications-service`).
- **Strategy Pattern** cuando haya variantes de reglas (promos, cobertura, retries).
- **Factory Pattern** solo si hay selección dinámica clara por tenant/proveedor.
- Evitar patrones innecesarios en MVP.

---

## 3) Convenciones técnicas (Java + Quarkus)

### 3.1 Versiones y librerías base
- Java 21
- Quarkus 3.x
- Jakarta Validation
- Jackson (JSON)
- JUnit 5 + Mockito

### 3.2 Convenciones de diseño
- Controllers/consumers sin lógica de negocio compleja.
- Reglas de negocio en `domain`/`application`.
- Mappers explícitos entre contratos y dominio.
- Métodos cortos y cohesivos.
- Nombres explícitos (evitar abreviaturas ambiguas).

### 3.3 Configuración y secretos
- Solo variables de entorno/config segura.
- Nunca hardcodear secretos.
- Nunca loguear tokens/firmas/payload sensible completo.

---

## 4) Reglas de APIs sync

- Endpoints bajo `/v1`.
- Validación inmediata de request.
- Timeouts explícitos para llamadas externas.
- Respuesta de error estandarizada.
- Operaciones críticas con `idempotencyKey`:
  - `POST /v1/orders`
  - `POST /v1/payments/checkout`

---

## 5) Reglas de flujos async

- Consumidores idempotentes por identificador de evento.
- Retries con backoff para errores transitorios.
- DLQ al agotar retries.
- Duplicados: responder/ack sin efectos secundarios.
- Publicación de eventos con envelope estándar (`schemaVersion`, `tenantId`, `correlationId`).

---

## 6) Estándar de observabilidad

### 6.1 Logs estructurados (mínimo)
- `timestamp`
- `level`
- `service`
- `operation`
- `tenantId`
- `correlationId`
- `status` (`success|error|duplicate|retry|dlq`)
- `message`

### 6.2 Métricas mínimas
- `http_requests_total`, `http_latency_ms`
- `events_processed_total`
- `events_duplicate_total`
- `events_retry_total`
- `events_dlq_total`
- `external_dependency_latency_ms`

---

## 7) Estándar de testing

### 7.1 Mínimos obligatorios por historia
- Happy path.
- Validaciones de input.
- Idempotencia.
- Error de dependencia externa.
- Duplicado (async).
- Firma inválida (webhook).

### 7.2 Regla de calidad
No se considera “done” una historia sin tests mínimos y sin logs estructurados.

---

## 8) Plantilla oficial para prompts de implementación (copiar/pegar)

```text
Implementa esta historia siguiendo de forma estricta:

1) docs/backlog/implementation-stories.md
2) docs/backlog/development-standards-and-stack-rationale.md

Restricciones obligatorias:
- Stack: Java 21 + Quarkus 3.x.
- Arquitectura: Hexagonal (application/domain/infrastructure).
- Contratos: usar exclusivamente ms-shared-contracts.
- tenantId y correlationId obligatorios.
- Manejo de errores estándar: {code, message, details}.
- Idempotencia obligatoria en operaciones críticas sync/async.
- Observabilidad mínima: logs estructurados + métricas base.
- Tests mínimos: happy path + validaciones + idempotencia + errores clave.
- No agregar features fuera del alcance MVP.

Entregables:
- Código
- Tests
- Documentación corta de ejecución/configuración
- Ejemplos JSON actualizados
```

---

## 9) Criterios de aceptación transversales (checklist de PR)

- [ ] ¿Usa Java 21 + Quarkus?
- [ ] ¿Respeta estructura hexagonal?
- [ ] ¿Importa contratos desde `ms-shared-contracts`?
- [ ] ¿Incluye `tenantId` y `correlationId` en los flujos?
- [ ] ¿Errores en formato estándar?
- [ ] ¿Idempotencia implementada?
- [ ] ¿Retries/DLQ donde aplica?
- [ ] ¿Logs y métricas mínimas implementadas?
- [ ] ¿Tests mínimos pasando?
- [ ] ¿Sin sobreingeniería ni scope creep?
