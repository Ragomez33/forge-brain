---
id: "TICK-001"
title: "Ausencia de autenticación y sesión en foundly-pos y desvinculación del Tenant/User de forge-backend"
author: "Brain-Corp Analyst"
target_domain: "shared"
status: "backlog"
created_at: "2026-11-07"
dependencies: []
github_issue: 1
---

## Contexto / Hallazgo

`foundly-pos` opera hoy como aplicación local-first sin ninguna capa de autenticación ni sesión remota. Su `package.json` no incluye cliente de Supabase ni librería de auth (solo `expo-sqlite`, `drizzle-orm`, `zustand`), no existe ningún módulo `auth` en `src/` y la única identidad de usuario es la tabla local `profile` en `sidecars/foundly-pos/src/db/schema.ts` (`id: integer`, `name`, `avatarUri`), gestionada por `src/modules/profile/services/profileService.ts` y `src/shared/hooks/useUserProfileStore.ts`.

En el lado `provider`, `forge-backend` sí expone un flujo completo de identidad: mutaciones `signUp`, `signIn`, `refreshSession`, `signOut` y `me` en `src/modules/auth/resolvers/auth.resolver.ts`, con JWT de Supabase verificado por JWKS (`src/shared/auth/AGENTS.md`) y claims obligatorios `sub` + `app_metadata.tenantId`. El modelo `User` (`prisma/schema.prisma`, sección IAM) usa `id` igual a `auth.users` de Supabase y `tenantId` como discriminante multi-tenant.

El vacío es bidireccional: el POS no puede obtener un JWT, no persiste tokens ni refresh, no resuelve su empresa local como `Tenant` del ERP y no tiene política de sesión offline/online. Además, `businesses.id` en el POS es `text` con valor por defecto `default-business` (`src/shared/constants/business.ts`) y se autogenera con `randomUUID`/fallback `biz_...` en `src/services/businessService.ts`, sin ninguna relación con el `Tenant.id` (UUID) del backend. Sin esta base, ningún envío de datos ni validación de licencia puede atribuirse a un tenant válido.

## Requerimientos Técnicos

- Definir el contrato de autenticación que consumirá el POS sobre las operaciones existentes de `forge-backend`: `signIn`, `refreshSession`, `signOut`, `me` y recuperación/verificación si aplica, con sus payloads (`AuthSession`, tokens, expiración) documentados en el contrato GraphQL.
- Definir el almacenamiento seguro de tokens en el POS (por ejemplo `expo-secure-store`) y el ciclo de vida: login, refresh automático, logout, expiración y renovación.
- Especificar el mapeo sesión→empresa local: cómo `app_metadata.tenantId` del JWT se vincula a una fila de `businesses` del POS, qué pasa en el primer login (onboarding/`createTenant`) y cómo se impide operar sin vinculación.
- Definir la política offline explícita: si el POS puede seguir operando sin red y por cuánto tiempo; cómo se firma/valida la sesión local y qué operaciones quedan bloqueadas.
- Especificar la sincronización de identidad de usuario (nombre, email, avatar) entre `User`/`me` del ERP y la tabla local `profile`, evitando dos fuentes de verdad divergentes.
- Criterios de aceptación: login exitoso produce sesión utilizable contra el backend; `tenantId` del JWT coincide con el `businessId` activo del POS; logout invalida tokens; sesión expirada no permite escrituras que deban sincronizarse.

## Impacto en la Arquitectura

Afecta a ambos sidecars (`foundly-pos` como `client` y `forge-backend` como `provider`) y establece la base de identidad sobre la que dependen los tickets de mapeo de datos, sincronización y licenciamiento. Riesgos: introducir dependencia de red en una app diseñada offline-first; divergencia entre `profile` local y `User` remoto; incompatibilidad entre `businesses.id` (`text`) y `Tenant.id` (`uuid`). Requiere contratos comunes de sesión y multi-tenancy. Es dependencia de `TICK-002`, `TICK-003` y `TICK-004`.
