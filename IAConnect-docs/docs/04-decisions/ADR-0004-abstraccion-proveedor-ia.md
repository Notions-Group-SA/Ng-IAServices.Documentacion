---
doc_id: ADR-0004
doc_type: adr
title: "ADR-0004: Abstracción de proveedor IA con factory por tenant"
status: draft
origin: reverse-engineered
confidence: high
date: 2026-07-15
deciders: [pendiente-asignacion]
owner: pendiente-asignacion
last_review: 2026-07-15
audience: [arquitectos, dev, agentes-automaticos]
classification: uso-interno
traces: [ADR-0003]
supersedes: null
---

# ADR-0004: Abstracción de proveedor IA con factory por tenant

> Decisión reconstruida. Confianza: alta (`Domain/Interfaces/IAIProvider.cs`, `Infrastructure/Providers/**`).

## Contexto

La solución debe soportar múltiples proveedores IA (Gemini, Claude, OpenAI) y permitir elegir el proveedor por
tenant sin cambiar el código de los casos de uso.

## Decisión

Definir el contrato `IAIProvider` en el dominio y una `IAIProviderFactory` que devuelve la implementación concreta
según `lut_Tenants.Proveedor_IA` (`switch(tenant.ProveedorIA.ToLower())`). El modelo, temperatura, tokens y API key
se toman **del tenant en BD**. Cada proveedor encapsula su transporte:
- **Claude:** `HttpClient` a `https://api.anthropic.com/` (`POST v1/messages`, headers `x-api-key` + `anthropic-version`), con retry propio.
- **Gemini:** SDK `Google_GenerativeAI`.
- **OpenAI:** SDK `OpenAI` (`ChatClient`).

## Alternativas

- **Un único proveedor fijo:** simple pero sin independencia de proveedor.
- **Gateway externo (p. ej. LiteLLM):** dependencia adicional; no adoptado.

## Consecuencias

- (+) Cambiar de proveedor es configuración del tenant, no código (patrón Factory/Strategy).
- (–) **Heterogeneidad de resiliencia:** solo Claude usa `IHttpClientFactory` + retry/timeout; OpenAI no tiene retry
  propio. Convendría unificar (Polly / `IHttpClientFactory` para todos).
- (–) La configuración `AIProviders:*` de `appsettings.json` (incl. `DefaultModel`) **no se consume**; el modelo
  efectivo sale del tenant. Config potencialmente confusa (gap `GAP-CONFIG-MODEL`).
