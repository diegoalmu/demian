# Arquitectura — Demian

## Visión general

Demian es un agente personal construido sobre OpenClaw, corriendo localmente en Endor PC (Windows 11 + WSL2/Ubuntu). Usa un patrón de **orquestador + sub-agentes**: un agente coordinador ligero que recibe los mensajes (Telegram / navegador) y delega el trabajo pesado a agentes especializados, cada uno con su propio dominio de memoria y permisos.

```
                    ┌─────────────────────┐
                    │   Telegram / Navegador│
                    │   (Cloudflare Tunnel) │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │   Gateway OpenClaw     │
                    │   (WSL2, Endor PC)      │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  Agente Orquestador    │
                    │  (routing, sin memoria │
                    │   propia extensa)       │
                    └──┬─────┬─────┬─────┬──┘
           ┌───────────┘     │     │     └───────────┐
           ▼                 ▼     ▼                 ▼
   ┌───────────────┐ ┌─────────────┐ ┌───────────┐ ┌─────────────┐
   │ Universidad/   │ │ Diario       │ │ DevOps    │ │ Web/         │
   │ Código          │ │ (Obsidian+GH) │ │ (GCP+OCI) │ │ Portafolio    │
   └───────────────┘ └─────────────┘ └───────────┘ └─────────────┘
```

## Componentes

### Gateway (OpenClaw)
Proceso central que multiplexa las conexiones de los canales (Telegram, navegador vía Cloudflare) y enruta hacia el runtime del agente. Corre como servicio systemd dentro de WSL2 para persistir entre reinicios.

### Agente Orquestador
No mantiene memoria extensa propia — su trabajo es interpretar la intención del mensaje entrante y delegar al sub-agente correcto. Usa un modelo ligero (local o cloud económico) para el triage, evitando gastar cómputo pesado en decidir a quién delegar.

### Sub-agentes

| Agente | Dominio | Skills principales | Memoria |
|---|---|---|---|
| Universidad/Código | Tareas de programación, debugging, análisis de PDFs de clase | ejecución de código, skills de PDF | `vault/universidad/` |
| Diario | Memoria personal, resúmenes, journaling | `obsidian`, `github` | `vault/diario/`, `vault/agent-memory/` |
| DevOps (fase 2) | Mantenimiento de bots en GCP y Oracle Cloud | `ssh-remote` (acceso restringido por host) | Configuración de VMs, logs |
| Web/Portafolio (fase 2) | Desarrollo del sitio web para el cliente/empresa | `github`, ejecución de código | `vault/proyectos/`, repo propio |

Cada sub-agente opera con **contexto y permisos aislados** — el de DevOps no tiene acceso al vault del diario, el de Universidad no tiene acceso SSH a las VMs. Esto limita el radio de daño si un solo agente es comprometido vía inyección de prompt.

## Estrategia de modelos

Dado que el hardware de Endor PC no tiene GPU dedicada, no se asume inferencia local pesada:

- **Triage / conversación casual**: modelo local ligero cuantizado (GGUF Q4_K_M) — rápido, bajo consumo de RAM.
- **Tareas que requieren razonamiento fuerte** (código, DevOps, redacción): modelo cloud bajo demanda (API de Anthropic, con Gemini Plus como proveedor alternativo configurado).
- Política explícita de qué tarea usa qué modelo, en vez de dejarlo a discreción del agente.

## Memoria

La memoria de largo plazo vive como **archivos Markdown planos** en un vault de Obsidian, no en una base de datos propietaria. Esto da:
- Legibilidad y edición humana directa.
- Versionado con Git (historial de cambios, posibilidad de revertir).
- Portabilidad — no depende de OpenClaw para poder leer la memoria acumulada.

El vault vive en el filesystem de Windows (accesible por Obsidian nativamente) y se lee desde WSL2 vía `/mnt/c/...`. Ver `docs/INSTALACION.md` para el detalle de sincronización.

## Acceso remoto

- **Telegram**: canal principal para uso diario/rápido, con allowlist restringida a un chat ID.
- **Cloudflare Tunnel**: expone el gateway a un dominio propio (`claw.tu-dominio.ejemplo`) sin abrir puertos en el router, protegido con Cloudflare Access (login obligatorio) y reglas de rate limiting.

## Consideraciones de seguridad en el diseño

El diseño asume que el agente **puede ser manipulado** vía contenido externo (mensajes, documentos, páginas web) — no es un supuesto teórico, hay ataques documentados contra OpenClaw en 2026. Por eso:
- Los sub-agentes están aislados por dominio y permisos.
- El agente DevOps, el de mayor riesgo, requiere confirmación explícita antes de ejecutar cambios.
- Contenido externo se etiqueta como no confiable antes de entrar al contexto del modelo.

Detalle completo en [`SEGURIDAD.md`](SEGURIDAD.md).
