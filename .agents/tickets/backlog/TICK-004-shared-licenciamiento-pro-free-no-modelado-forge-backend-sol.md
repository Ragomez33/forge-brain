---
id: "TICK-004"
title: "Licenciamiento Pro/Free no modelado: forge-backend solo expone activeModules y foundly-pos solo feature flags locales"
author: "Brain-Corp Analyst"
target_domain: "shared"
status: "backlog"
created_at: "2026-11-07"
dependencies: ["TICK-001"]
github_issue: 4
---

## Contexto / Hallazgo

No existe en `forge-backend` un modelo de plan, suscripción o licencia. `prisma/schema.prisma` no contiene entidades `Plan`, `Subscription`, `License` ni campos de plan/trial en `Tenant`; el único mecanismo de habilitación es `TenantSettings.activeModules String[]` (relación `Tenant.entitlements`), validado por `EntitlementGuard` (`src/shared/auth/entitlement.guard.ts`) mediante `@RequireFeature` contra el enum `FeatureModule` (`IAM`, `INVENTORY`, `SALES`, `CASH_REGISTER`, `CONTACTS`, `ACCOUNTING`, `PURCHASES` en `src/shared/enums/feature-module.enum.ts`). La query `activeModules` (`src/modules/iam/resolvers/tenant-settings.resolver.ts`) requiere usuario autenticado y devuelve solo la lista de módulos, no un plan ni estado de suscripción. Las mutaciones `activateModule`/`deactivateModule` exigen `iam:settings:manage`; no hay flujo de compra, renovación ni expiración.

En el lado `client`, `foundly-pos` gestiona capacidades con una tabla local `feature_flags` (`src/db/schema.ts`) y `src/services/featureFlagService.ts`, con `FUNCIONALIDAD_BIMONEDA` y `FUNCIONALIDAD_COBRO_MULTIPLE_TASA` por defecto en `false` y conmutables localmente vía `setFlag`. No hay chequeo periódico contra el backend, ni concepto de plan Pro/Free, ni expiración o degradación. El POS tampoco sabe qué `FeatureModule` del ERP corresponde a cada flag.

El vacío es doble: el ERP no expone un contrato de licenciamiento (plan/estado/vigencia) y el POS no consume entitlements remotos ni muta su comportamiento según ellos.

## Requerimientos Técnicos

- Definir el modelo de licenciamiento en `forge-backend`: campo/enum de plan en `Tenant` (p. ej. `plan: FREE | PRO`), o entidades `Subscription`/`Plan` con vigencia (`status`, `validUntil`, `graceUntil`), y su relación con `TenantSettings.activeModules`.
- Definir la regla de derivación plan→entitlements: qué `FeatureModule` habilita cada plan y cómo se materializan en `activeModules` (o se calculan en lectura) sin ambigüedad.
- Exponer una query autenticada de estado de licencia (p. ej. `subscriptionStatus` o `tenantPlan`) que devuelva plan, módulos activos, vigencia y estado (`ACTIVE`/`EXPIRED`/`GRACE`), respetando el `tenantId` del JWT.
- Definir en el POS la estrategia de chequeo periódico: intervalo, caché local del último estado válido, comportamiento offline y periodo de gracia antes de degradar.
- Especificar la mutación de comportamiento: mapeo explícito `FeatureModule`/plan → flags del POS (`FUNCIONALIDAD_BIMONEDA`, `FUNCIONALIDAD_COBRO_MULTIPLE_TASA`) y a funcionalidades Pro adicionales; qué se bloquea, qué se muestra en solo lectura y qué mensajes de upgrade se presentan.
- Definir política fail-safe: qué ocurre si el backend no responde o devuelve error (conservar último estado vs. degradar a Free) y evitar que una licencia vencida permita escrituras indebidas.
- Criterios de aceptación: un tenant Free recibe módulos/flags de Free; un tenant Pro los de Pro; un cambio de plan se refleja tras el siguiente chequeo; un tenant sin red conserva el último estado dentro de la gracia y luego degrada de forma determinista.

## Impacto en la Arquitectura

Afecta a `forge-backend` (modelo de plan/suscripción, query de licencia, posible integración de billing) y a `foundly-pos` (servicio de licencia, mapeo de entitlements a flags, UI de upgrade y bloqueo de funciones). Riesgos: acoplar el POS a una topología de módulos que no representa planes comerciales; inconsistencias de caché/TTL; degradación abrupta en modo offline; habilitar funciones Pro por manipulación de la tabla `feature_flags` local. Depende de `TICK-001` para autenticar el tenant; se recomienda coordinar con `TICK-002` para resolver `business_id ↔ Tenant.id`.
