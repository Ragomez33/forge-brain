---
id: "TICK-002"
title: "Falta de mapeo de identidad y modelos entre el esquema SQLite de foundly-pos y el esquema Prisma/Supabase de forge-backend"
author: "Brain-Corp Analyst"
target_domain: "shared"
status: "backlog"
created_at: "2026-11-07"
dependencies: ["TICK-001"]
github_issue: 2
---

## Contexto / Hallazgo

Los dos esquemas no comparten identificadores ni representación de datos, lo que impide hoy cualquier integridad referencial entre ventas/pagos/clientes locales y las tablas del ERP central.

Evidencia en `sidecars/foundly-pos/src/db/schema.ts`:
- `businesses.id` es `text` (default `default-business`); `products`, `customers`, `sales`, `saleItems`, `payments`, `cashMovements`, `inventoryMovements`, etc. usan `integer('id').primaryKey({ autoIncrement: true })`.
- Los montos son enteros en centavos: `priceCents`, `costPriceCents`, `totalCents`, `amountPaidCents`, `amountCents`, `unitPriceCents`, `appliedRate`.
- Los timestamps son `text` ISO y hay enums propios: `paymentType` (`cash`|`credit`), `paymentMethods.currency` (`USD`|`VES`|`USDT`), `payments.method` legacy (`cash`|`card`|`transfer`|`other`).
- `customers` (`id` integer, `name`, `phone`, `documentId`) no equivale a `Partner`.

Evidencia en `sidecars/forge-backend/prisma/schema.prisma`:
- `Tenant.id`, `Partner.id`, `ProductTemplate.id`, `ProductVariant.id`, `SalesOrder.id`, `SalesOrderLine.id`, `Payment.id`, `AccountingInvoice.id` son `String @id @default(uuid())`, con `tenantId: String` como discriminante.
- Los importes son `Decimal @db.Decimal(12, 4)` (`SalesOrder.total`, `Payment.amount`, `amountUsd`, etc.), no centavos enteros.
- Las ventas se modelan como `SalesQuote`/`SalesOrder` (con `status: SalesDocumentStatus`, `variantId`, `taxableBase`, `vatTotal`, `igtfAmount`) y los cobros como `Payment` contable con `idempotencyKey` y `PaymentAllocation`; no existe una entidad `Sale` equivalente al `sales` local.
- Enums divergentes: `Currency` (`USD`/`VES`), `PaymentMethod` (`TRANSFER`|`MOBILE_PAYMENT`|`CASH`|`ZELLE`|`CHECK`|`OTHER`), `SalesDocumentStatus` (`DRAFT`|`APPROVED`|`DONE`|`CANCELLED`).

No existe en el POS ninguna columna `remote_id`/`external_id`, tabla de correlación ni generación de UUID para ventas, pagos o clientes. Tampoco hay equivalencia entre `products` (SKU con stock/centavos) y `ProductVariant`/`ProductTemplate` del ERP.

## Requerimientos Técnicos

- Definir una estrategia de identidad global: generar UUID en origen (POS) para `sales`, `saleItems`, `payments` y `customers`, o una tabla de correlación local `local_id ↔ remote_uuid` por entidad, garantizando idempotencia y trazabilidad.
- Especificar el mapeo `businesses.id (text) ↔ Tenant.id (uuid)` y el orden de resolución padre→hijo para respetar FKs: Tenant → Partner/Customer → ProductVariant → SalesOrder/SalesOrderLine → Payment/PaymentAllocation.
- Definir el mapeo de dinero: centavos enteros (`integerCents`) ↔ `Decimal(12,4)`; reglas de redondeo, escala y precisión; mapeo FX (`appliedRate`, `appliedRateType`, `totalAmountVesCents`, `fxAdjustmentCents`) a `exchangeRate`/`rateSource`/`totalVes`/`appliedUsd`/`appliedVes`.
- Definir tablas de traducción de enums: `paymentType` ↔ `SalesDocumentStatus`/estado de cobro; `payments.method` y `payment_methods.name` ↔ `PaymentMethod`; `currency` ↔ `Currency`; `cashMovements.category` ↔ módulo de caja/contabilidad.
- Definir el mapeo de `products` (SKU, `stock`, `priceCents`) a `ProductTemplate`/`ProductVariant` y `StockQuant`/`StockMove`, incluyendo unidad y ubicación.
- Especificar qué entidades locales NO se sincronizan (por ejemplo `featureFlags`, `appSettings`, `profile`) y cuáles son de solo lectura desde el ERP.
- Criterios de aceptación: un registro local puede resolverse de forma determinista a su contraparte remota; una venta con sus líneas y pagos se reconstruye sin violar FKs; los montos conservan valor exacto tras round-trip.

## Impacto en la Arquitectura

Es el contrato de datos central entre ambos sidecars y bloquea la sincronización (`TICK-003`) y el licenciamiento (`TICK-004`). Riesgos: pérdida de integridad por IDs autoincrementales no globales; descuadres monetarios por centavos vs `Decimal(12,4)`; divergencia de enums y estados de venta; duplicados por falta de idempotencia. Depende de `TICK-001` para conocer el `tenantId` y la identidad de sesión.
