# Demian

Agente personal self-hosted construido sobre [OpenClaw](https://github.com/openclaw/openclaw), con arquitectura de orquestador + sub-agentes especializados, memoria persistente en Obsidian sincronizada a GitHub, y acceso remoto vía Telegram y navegador (Cloudflare Tunnel).

Corre en hardware local (Endor PC, Windows 11 + WSL2) sin depender de servicios de hosting de terceros para el runtime del agente.

## Características

- **Orquestador + sub-agentes**: cada dominio de tarea (universidad, diario, infraestructura, proyectos web) tiene su propio agente con memoria y permisos separados.
- **Memoria en Markdown**: el "cerebro" del agente vive en un vault de Obsidian versionado en Git — legible, editable y auditable por humanos, no una base de datos opaca.
- **Acceso multi-canal**: Telegram para uso diario, navegador vía Cloudflare Tunnel para trabajo desde cualquier computadora.
- **Diseñado para hardware limitado**: estrategia explícita de router de modelos (local ligero vs. cloud bajo demanda) en lugar de asumir GPU disponible.
- **Seguridad tratada como requisito, no como nota al pie**: rate limiting, allowlists, defensas contra inyección de prompts, y control de acceso por sub-agente.

## Arquitectura

Ver [`ARQUITECTURA.md`](ARQUITECTURA.md) para el detalle de cómo se comunican el orquestador y los sub-agentes, y qué hace cada uno.

## Instalación

Ver [`INSTALACION.md`](INSTALACION.md) para el paso a paso completo (WSL2, OpenClaw, Telegram, Cloudflare Tunnel).

## Seguridad

Ver [`SEGURIDAD.md`](SEGURIDAD.md) — política de accesos, defensas contra prompt injection y abuso, y checklist de mantenimiento.

## Plan de acción

Ver [`PLAN-DE-ACCION.md`](PLAN-DE-ACCION.md) para el roadmap completo por fases, con estado de avance.

## Estructura del repo

```
demian/
├── README.md
├── .env.example          # variables de entorno de ejemplo (sin valores reales)
├── .gitignore
├── ARQUITECTURA.md
├── INSTALACION.md
├── SEGURIDAD.md
├── PLAN-DE-ACCION.md
└── BITACORA-INSTALACION.md
```

> Nota: el vault de Obsidian (diario, notas de universidad) vive en un repositorio **privado separado**, no en este repo. Este repo es la documentación pública/portafolio del proyecto de infraestructura del agente.

## Estado del proyecto

🟡 En fase de planeación — instalación aún no iniciada.

## Licencia

MIT — ver [`LICENSE`](LICENSE).
