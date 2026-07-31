---
doc_id: ADR-0005
doc_type: adr
title: "ADR-0005: Autenticación JWT con refresh tokens y bloqueo por intentos"
status: draft
origin: reverse-engineered
confidence: high
date: 2026-07-15
deciders: [pendiente-asignacion]
owner: pendiente-asignacion
last_review: 2026-07-15
audience: [arquitectos, dev, seguridad, agentes-automaticos]
classification: uso-interno
traces: []
supersedes: null
---

# ADR-0005: Autenticación JWT con refresh tokens y bloqueo por intentos

> Decisión reconstruida. Confianza: alta (`Application/Services/AuthService.cs`, `API/Program.cs`, `sys_Refresh_Tokens`).

## Contexto

La API es *stateless* y multi-cliente; necesita autenticación estándar, renovación de sesión sin re-login y defensa
básica ante fuerza bruta.

## Decisión

Emitir **JWT** (firma HmacSha256; issuer/audience/lifetime validados, `ClockSkew=0`) con expiración corta
configurable por tenant, más **refresh tokens** (64 bytes aleatorios, rotación, revocables, tabla `sys_Refresh_Tokens`).
Contraseñas con **BCrypt**. **Bloqueo tras 5 intentos fallidos durante 15 minutos** (`AccountLockedException`→423).

## Alternativas

- **Sesiones con estado en servidor:** no encaja con API stateless multi-cliente.
- **OAuth2/OIDC con IdP externo:** más robusto y estándar; mayor complejidad; no adoptado (posible evolución).

## Consecuencias

- (+) Autenticación estándar, renovación sin re-login, revocación de refresh tokens, mitigación de fuerza bruta.
- (–) La expiración/rotación depende de la config por tenant; el secreto JWT debe venir de gestor de secretos
  (los defaults de dev no sirven en producción).
- (–) Sin IdP externo: la gestión de identidades vive en la propia solución.
