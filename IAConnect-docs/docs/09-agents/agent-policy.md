---
doc_id: AGENT-POL-001
doc_type: agent-policy
title: Política de agentes documentales — IAConnect
version: 1.0.0
status: draft
origin: human
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 365
audience: [agentes-automaticos, dev, arquitectos]
classification: uso-interno
traces: []
supersedes: null
---

# Política de agentes documentales — IAConnect

> Define el perímetro de acción de cualquier agente que genere o actualice esta documentación (Marco §12.4).

1. Los agentes crean y actualizan documentos **solo** en `status: draft`.
2. La promoción `draft → approved` la ejecuta un **humano** (revisor registrado en el PR).
3. Los agentes **nunca** modifican: el código fuente de `/NG/Ng-IAServices`, ninguna base de datos, ADRs
   `approved/superseded`, changelogs publicados, evidencia de auditoría, ni `GLOSSARY.md` sin revisión.
4. **Sin secretos ni PII** en ningún entregable: credenciales, API keys, cadenas de conexión y datos reales quedan
   fuera; los ejemplos se sintetizan o seudonimizan.
5. Ante **contradicción entre fuentes**, el agente la **reporta** (no elige en silencio). Ante código↔doc previa,
   **gana el código**.
6. Contenido inferido lleva `origin` y `confidence` en el frontmatter.
7. Toda escritura ocurre **solo** bajo `Ng-IAServices.Documentacion/`; el origen es de solo lectura.
8. El estado documental es calculable por máquina desde [`docs-manifest.yaml`](../../docs-manifest.yaml) + frontmatter.

## Cómo un agente recupera contexto

1. Leer la ia-db: [`ia-db/README.md`](../../../ia-db/README.md) → índice relevante.
2. Ampliar a la doc de pieza o al código **solo** ante insuficiencia comprobada (Marco/`Rule-Indexing`).
