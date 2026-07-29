# Bitácora de Instalación — Demian

> Registro cronológico real de la instalación (no el plan ideal, sino lo que pasó de verdad, incluyendo errores y soluciones). Complementa a `PLAN-DE-ACCION.md`.

**Entorno:** Windows 11 + WSL2/Ubuntu, sin GPU dedicada. Usuario Linux: `endoralex`, máquina: `endorpc`.

---

## ✅ Completado

### 1. Virtualización y WSL2
- Virtualización confirmada activa en BIOS/Administrador de Tareas.
- `wsl --install` ejecutado desde PowerShell como administrador.
- Ubuntu instalado, usuario Linux `endoralex` creado.
- Confirmado WSL2 (no WSL1) con `wsl -l -v` desde PowerShell.

### 2. systemd habilitado
- Al hacer `sudo apt update && sudo apt upgrade -y` apareció el error esperado: `Failed to connect to system scope bus` — normal en WSL2 sin systemd activado.
- Solución: editar `/etc/wsl.conf` con `sudo nano /etc/wsl.conf`, agregar:
  ```
  [boot]
  systemd=true
  ```
- Reinicio completo de WSL2 desde PowerShell: `wsl --shutdown`, esperar, reabrir Ubuntu.
- Confirmado con `systemctl status` → `State: running`.

### 3. Instalación de OpenClaw
- `curl -fsSL https://openclaw.ai/install.sh | bash`
- `openclaw onboard --install-daemon` → instaló el Gateway como servicio systemd de usuario.
- Se habilitó "lingering" automáticamente (necesario para que el servicio siga corriendo tras cerrar sesión).
- **Gateway confirmado corriendo en `ws://127.0.0.1:18789`** — puerto real verificado, coincide con lo documentado en `INSTALACION.md`.
- Workspace: `~/.openclaw/workspace`.

### 4. Problema de PATH (resuelto)
- `openclaw` no se reconocía como comando fuera de la ruta completa.
- Diagnóstico: `npm prefix -g` → `/home/endoralex/.npm-global`.
- Solución permanente:
  ```bash
  echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
  source ~/.bashrc
  ```
- Confirmado con `openclaw --version` → `OpenClaw 2026.7.1-2 (0790d9f)`.

### 5. Configuración de proveedor de modelo — OpenRouter (con fricciones)
- Cuenta de OpenRouter creada, API key generada.
- Primer intento: modelo escrito manualmente como `openai/gpt-oss-20b:free` → **error**: quedó registrado bajo el proveedor `openai` en vez de `openrouter` (`Unknown model` al probarlo).
- Segundo intento: reconfigurar con `openclaw onboard` → `configure a model provider` → OpenRouter → esta vez ofreció **OAuth** en vez de API key directa.
  - El callback automático de OAuth falló (`ERR_CONNECTION_REFUSED` en `localhost:3000`) — fricción conocida de OAuth cruzando Windows↔WSL2.
  - Solución: copiar la URL completa del navegador (con el `code=...`) y pegarla manualmente cuando el wizard pidió `Paste the OpenRouter redirect URL`.
- Al llegar a "Browse all models", el buscador no encontraba `gpt-oss-20b:free` con el prefijo correcto — se resolvió escribiendo manualmente el ID completo: `openrouter/openai/gpt-oss-20b:free`. Esta vez sí quedó registrado bajo el proveedor correcto.
- **Primera prueba real** (`talk to agent`) → error: `provider rejected the request schema or tool payload`. Causa probable: `gpt-oss-20b` tiene soporte de "tool calling" poco confiable en varios reportes de la comunidad, aunque en teoría lo soporta.
- Se intentó cambiar a `openrouter/poolside/laguna-m.1:free` (recomendado para tool calling) → error: `The selected model was not found by the provider` (ID no coincidía con el catálogo real en ese momento — el catálogo de gratis de OpenRouter cambia constantemente).

**Decisión:** en lugar de seguir iterando con IDs de OpenRouter inestables, se cambia de estrategia a **Ollama local** como proveedor principal para las pruebas iniciales.

---

## ✅ Completado (continuación — 29 de julio)

### 6. Instalación de Ollama
- `curl -fsSL https://ollama.com/install.sh | sh` → falló la primera vez: `ERROR: This version requires zstd for extraction`.
- Solución: `sudo apt-get install zstd -y`, luego reintentar la instalación.
- `sudo systemctl enable --now ollama` → activo y persistente.
- `ollama pull llama3.2:3b` → descargado con éxito (2GB).

### 7. Auditoría de seguridad
- `openclaw security audit` detectó 3 warnings, el principal: `gateway.controlUi.allowInsecureAuth=true`.
- Corregido editando `~/.openclaw/openclaw.json` → `"allowInsecureAuth": false`, reinicio del Gateway.
- Quedó 1 solo warning restante (`gateway.trusted_proxies_missing`) — **no urgente ahora, pendiente para cuando se configure Cloudflare Tunnel** (hay que setear `gateway.trustedProxies` con las IPs del proxy en ese momento).

### 8. Mitigación de crash-loop de Ollama en WSL2
- OpenClaw detectó automáticamente el riesgo: `ollama.service` con `Restart=always` + CUDA visible (aunque no hay GPU real) puede pinear memoria y provocar que Windows reinicie la VM de WSL2 completa.
- Mitigación aplicada (la menos invasiva de las 3 sugeridas): `sudo systemctl edit ollama` → agregar `Environment="OLLAMA_KEEP_ALIVE=5m"` → `daemon-reload` + `restart ollama`.
- Referencia: https://github.com/ollama/ollama/issues/11317

### 9. Configuración de Ollama como proveedor en OpenClaw
- `openclaw onboard` → `configure a model provider` → **Ollama** → modo **Local only** (no "Cloud + Local" ni "Ollama Cloud", para no depender de nada externo).
- Base URL: `http://127.0.0.1:11434` (default detectado automáticamente, sin modificar).
- Modelo seleccionado del catálogo real: `ollama/llama3.2:3b` — a diferencia de OpenRouter, Ollama sí lista correctamente los modelos instalados localmente, sin necesidad de adivinar IDs.

### 10. Error de sesión colgada (resuelto — no era problema de modelo)
- Al escribir `talk to agent`, apareció: `Which agent would you like to talk to?` repetidamente sin importar la respuesta.
- Se intentó seguir la sugerencia literal del propio Crestodian ("switch to your agent TUI"), pero esto llevó a un comportamiento errático — el modelo empezó a inventar pasos de configuración que no correspondían (reconfigurar autenticación, volver a preguntar por proveedor ya configurado). **Lección: cuando un asistente en modo determinístico empieza a "alucinar" pasos que no tienen sentido con lo ya configurado, mejor cortar y reiniciar limpio que seguirle la corriente.**
- Causa raíz real (encontrada en los logs vía `journalctl` y el archivo `/tmp/openclaw/openclaw-*.log`): esa sesión vieja de Crestodian tenía una restricción `runtime toolsAllow: crestodian`, pero el perfil de herramientas activo (`tools.profile: coding`) eliminaba esa misma herramienta de la lista disponible — quedando con **cero herramientas utilizables**, de ahí el error `No callable tools remain after resolving explicit tool allowlist`. No tenía relación con el modelo (pasó igual con OpenRouter y con Ollama).
- Solución: abandonar esa sesión colgada, usar directamente `openclaw tui` (entra derecho al agente `main`, sin pasar por Crestodian) y reiniciar el Gateway limpio con `systemctl --user restart openclaw-gateway`.

### 🎉 Hito: primera respuesta exitosa del agente
```
hola, ¿estás funcionando?
I'm functioning properly. How can I assist you today?
```
Con `ollama/llama3.2:3b`, 100% local, ~40 segundos de respuesta (esperado en CPU sin GPU dedicada). Confirma el flujo completo: WSL2 → systemd → OpenClaw Gateway → Ollama → respuesta del modelo.

---

## 🔄 En progreso

### Pendiente:
- [ ] Conectar Telegram como canal de acceso
- [ ] Evaluar si conviene agregar una GPU a Endor PC para modelos más grandes (pendiente de confirmar tipo de gabinete/PSU)
- [ ] Configurar Cloudflare Tunnel + resolver el warning de `trusted_proxies_missing` en ese momento
- [ ] Definir vault de Obsidian y su ruta de sincronización

---

## Lecciones aprendidas (para no repetir)

1. **Nunca escribir un ID de modelo "a ojo"** sin el prefijo del proveedor (`openrouter/...`) — causa el error de "Unknown model" registrado bajo el proveedor equivocado.
2. **El catálogo de modelos gratis de OpenRouter cambia constantemente** — un ID que funcionaba hace días puede dar `model not found`. Verificar en `openrouter.ai/models` antes de asumir.
3. **OAuth entre Windows y WSL2 tiene fricción conocida** con el callback local — la solución manual (pegar la URL completa) es confiable, mejor recordar este paso que perder tiempo reintentando el automático.
4. **No todos los modelos "gratis" son buenos para tool calling** — algunos están pensados solo para chat conversacional y fallan al usar herramientas del agente. Priorizar los que se anuncian explícitamente para uso agéntico.
5. **`sudo apt update && upgrade` en WSL2 recién instalado casi siempre requiere habilitar systemd después** — no es un error real, es un paso esperado del proceso.
