---
id: "TICK-003"
title: "Sincronización batch POS→ERP inexistente: falta outbox local y endpoint de ingesta en forge-backend"
author: "Brain-Corp Analyst"
target_domain: "shared"
status: "backlog"
created_at: "2026-11-07"
dependencies: ["TICK-001", "TICK-002"]
github_issue: 3
---

## Contexto / Hallazgo

`foundly-pos` no tiene capa de sincronización. Sus escrituras son transacciones locales síncronas sobre SQLite: `checkout()` en `src/modules/pos/services/posService.ts` inserta `sales`, `saleItems`, `cashMovements` e `inventoryMovements` dentro de `db.transaction(...)`, y `registerPayment()` en `src/modules/receivables/services/receivablesService.ts` hace lo propio. No existe tabla `outbox`, cola, reintentos, estado de envío ni cliente HTTP hacia el ERP. La única llamada de red del POS es la consulta de tasas en `src/services/exchangeRateService.ts` (`fetch` a BCV/Binance), ajena a la sincronización de negocio.

En `forge-backend` no existe una mutación de ingesta por lote para ventas/pagos externos. La superficie de ventas (`src/modules/sales/resolvers/sales-order.resolver.ts`) expone operaciones unitarias `createSalesOrder`, `approveSalesOrder`, `dispatchSalesOrder`, `cancelSalesOrder`, `salesOrder`, `salesOrders`, y los cobros viven en el módulo contable. Los `PurchaseOutboxEvent`/`InventoryOutbox` son mecanismos internos de otros módulos, no un punto de entrada para un POS. La tabla `Payment` sí tiene `idempotencyKey` con unicidad por tenant (`@@unique([tenantId, idempotencyKey])`), pero no hay contrato que la use para lotes externos.

## Requerimientos Técnicos

- Diseñar una tabla local de outbox en el POS (por ejemplo `sync_outbox`) con: `id`, `entity_type`, `entity_id`, `operation` (create/void), `payload`, `status` (pending/sent/failed), `attempts`, `last_error`, `created_at`, `sent_at`, y clave de idempotencia estable por registro.
- Definir la unidad de lote: agrupación por tenant y por secuencia de eventos, con orden padre→hijo (venta antes que líneas/pagos) y ventana de reintento con backoff.
- Especificar una mutación de ingesta en `forge-backend` (p. ej. `syncPosBatch(input)`) autenticada por el JWT del POS, que reciba lotes heterogéneos, valide el `tenantId` del contexto y devuelva resultados por ítem (aceptado, duplicado/idempotente, rechazado con código).
- Definir idempotencia extremo a extremo: clave por registro local enviada al backend, deduplicación en `Payment.idempotencyKey`/`SalesOrder` y respuesta idempotente ante reintentos.
- Especificar el manejo de fallos parciales y conflicto: qué ocurre si una venta se anula (`voidSale`) antes de sincronizarse; cómo se reconcilian ventas ya enviadas y luego anuladas; política de bloqueo o compensación.
- Definir límites del lote (tamaño máximo, timeouts), paginación y confirmación (ack) que permita marcar el outbox como enviado solo tras respuesta exitosa.
- Criterios de aceptación: corte de red durante un lote no pierde ni duplica datos; reintento produce el mismo resultado; el backend rechaza tenant distinto al del JWT; el estado local refleja pendientes/enviados/fallidos.

## Impacto en la Arquitectura

Cruza `foundly-pos` (outbox, worker de sincronización, resolución de conflictos) y `forge-backend` (nueva mutación de ingesta, validación de tenant, idempotencia, mapeo a `SalesOrder`/`Payment`). Riesgos: duplicados por reintentos sin idempotencia; violación de FKs por orden de envío; crecimiento no acotado del outbox; exposición de datos de un tenant a otro si el `tenantId` se toma del payload. Depende de `TICK-001` (sesión/JWT) y `TICK-002` (mapeo de IDs y modelos).
