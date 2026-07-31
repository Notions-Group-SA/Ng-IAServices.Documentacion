# Ng-IAServices.Documentacion

Conjunto documental de la solución **IAConnect** (`/NG/Ng-IAServices`), generado conforme al
[Marco de Documentación de Soluciones de Software](../../IA.Prompting.Templates/Referencias/Marco-Documentacion-Software-v1.md)
mediante el Tool-Prompt `Documentar-Fuentes-Software.md` / Profile `Solution-Documentation`.

> **Estado:** `draft` — reconstruido por ingeniería inversa desde el código fuente. Pendiente de revisión humana
> y promoción a `approved`. El origen (`/NG/Ng-IAServices`) **no fue modificado**.

## Cómo navegar

| Necesitás… | Ir a |
|---|---|
| Entender qué es la solución y cómo se organiza la doc | [`IAConnect-docs/README.md`](IAConnect-docs/README.md) — índice maestro |
| Recuperar contexto rápido sin releer el código (para agentes IA) | [`ia-db/README.md`](NG/Ng-IAServices.Documentacion/ia-db/README.md) — base de conocimiento |
| Calcular el gap documental (máquina) | [`IAConnect-docs/docs-manifest.yaml`](IAConnect-docs/docs-manifest.yaml) |
| Vocabulario controlado | [`IAConnect-docs/GLOSSARY.md`](IAConnect-docs/GLOSSARY.md) |

## Estructura

```text
Ng-IAServices.Documentacion/
├── README.md              ← este archivo
├── ia-db/                 ← base de conocimiento indexada (Knowledge-Indexing)
└── IAConnect-docs/        ← conjunto documental (Marco §5.2, adaptado)
    ├── README.md          ← índice maestro
    ├── GLOSSARY.md
    ├── docs-manifest.yaml
    └── docs/              ← 00-overview, 01-architecture, 02-requirements,
                             03-data, 04-decisions, 05-apis, 07-operations,
                             08-onboarding, pieces/
```

## Advertencia de seguridad

Ningún entregable de esta documentación contiene credenciales, cadenas de conexión con secretos ni datos
personales reales. Los ejemplos usan datos **sintéticos**. Las claves de API de los proveedores IA y las
cadenas de conexión reales viven en configuración/gestor de secretos del origen, fuera de esta documentación.
