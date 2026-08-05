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

## ✅ Completado (continuación — 31 de julio)

### 11. Decisión: modelos locales (Ollama) no aptos como agente conversacional principal
Tras varios intentos con `llama3.2:3b` y `qwen2.5:3b` (y evaluar subir a `qwen2.5:7b`), se confirmó un patrón consistente:
- Respuestas incoherentes (mezcla de idiomas invertida — respondía en inglés cuando se le escribía en español y viceversa).
- No seguía instrucciones persistentes (ni vía `USER.md`, ni vía instrucción directa en conversación).
- Loops de herramientas incluso con perfiles restringidos (`coding`, luego `messaging`).
- Tiempos de respuesta de 40 segundos a varios minutos, sin GPU dedicada.
- La propia documentación oficial de OpenClaw lo confirma: *"For best quality and security, use the strongest latest-generation model available."*

**Decisión:** Ollama se mantiene instalado pero **detenido por defecto** (no se usa como agente principal). Posible uso futuro: modo offline de emergencia, o router de triage si se suman más sub-agentes. El agente conversacional principal pasa a usar **Gemini 2.5 Flash** (cuenta Gemini Plus ya pagada).

Para detener Ollama sin desinstalarlo:
```bash
sudo systemctl disable ollama
sudo systemctl stop ollama
```

### 12. Configuración de Gemini como modelo principal
- `openclaw onboard` → `configure a model provider` → **Google** → autenticación con cuenta de Google.
- Modelo seleccionado: `gemini-2.5-flash`.
- **Mismo error de prefijo de proveedor que ya habíamos visto con OpenRouter**: el modelo quedó registrado como `openai/gemini-2.5-flash` en vez de `google/gemini-2.5-flash` — causaba `Unknown model: openai/gemini-2.5-flash`.
- Solución:
  ```bash
  openclaw config set agents.defaults.model.primary "google/gemini-2.5-flash"
  ```
- El cambio se guardó correctamente en el archivo (`grep -n "primary" ~/.openclaw/openclaw.json` lo confirmó), pero **`systemctl --user restart` no fue suficiente** para que el Gateway recargara el nuevo modelo — hizo falta un stop/start completo:
  ```bash
  systemctl --user stop openclaw-gateway
  sleep 3
  systemctl --user start openclaw-gateway
  ```
- Confirmación de que el modelo correcto cargó, revisando el log:
  ```bash
  journalctl --user -u openclaw-gateway --since "15 minutes ago" --no-pager | grep -i "agent model"
  ```
  → `agent model: google/gemini-2.5-flash` ✅

### 🎉 Hito: agente funcionando correctamente con Gemini
Con sesión nueva (`/new`) dentro de `openclaw tui`, el agente respondió de forma coherente, en español, con descripción precisa de sus propias capacidades (archivos, navegador, cron, código, memoria vía `MEMORY.md`). Tiempo de respuesta: segundos, no minutos.
- Se le asignó nombre: **Demian**.

### Lección aprendida (patrón repetido)
El error `Unknown model: <provider>/<modelo>` con el proveedor equivocado ya se presentó **dos veces** (con OpenRouter al inicio, y ahora con Google/Gemini). Causa común: al configurar un modelo por primera vez con un proveedor nuevo, el wizard a veces lo registra bajo un proveedor genérico por defecto en vez del proveedor real seleccionado. **Solución confiable en ambos casos**: corregir manualmente con `openclaw config set agents.defaults.model.primary "<proveedor-correcto>/<modelo>"`, seguido de un **stop/start completo del Gateway** (no solo `restart`).

### Nota sobre OpenCode (herramienta evaluada, no adoptada aún)
Se investigó OpenCode (agente de código para terminal, tipo Claude Code, por SST) como posible herramienta adicional. Confirmado por documentación oficial: funciona tanto en Windows nativo como en WSL2, pero **se recomienda WSL2 para mejor rendimiento y compatibilidad completa**. No es un reemplazo de OpenClaw — son herramientas de propósitos distintos (OpenCode: programación en terminal; OpenClaw: gateway de mensajería/agente personal). Podrían convivir en el mismo entorno WSL2 sin conflicto. Pendiente decidir si se integra al proyecto.

### ⚠️ Nota pendiente de revisar — rendimiento de archivos en `/mnt/c/`
Se encontró una advertencia de la comunidad: guardar datos de agentes en `/mnt/c/...` (acceso a Windows desde WSL2) puede ser **10-20 veces más lento** que usar el filesystem nativo de Linux (`/home/usuario/`). Esto afecta directamente la decisión tomada en la Fase 3 del plan (vault de Obsidian en `/mnt/c/Users/...`). **Pendiente evaluar** si conviene migrar el vault a un directorio nativo de WSL2 más adelante, considerando el trade-off con la comodidad de edición directa desde Obsidian en Windows.

---

## ✅ Completado (continuación — 31 de julio, tarde)

### 13. Migración del vault: de Google Drive a disco local
- Vault original vivía en `G:\Mi unidad\OBSIDIAN\` — unidad **virtual** creada por Google Drive en modo "Transmitir archivos" (Stream), **no accesible desde WSL2** (`ls /mnt/` solo mostraba `c`, `wsl`, `wslg`; confirmado también que no aparece como unidad de red vía `net use`).
- Decisión: eliminar la capa de Google Drive por completo — ya se tiene sincronización real y versionada vía Git/GitHub Desktop, que es redundante con Drive y más simple sin él.
- Migración manual: contenido movido de `G:\Mi unidad\OBSIDIAN\` a `C:\Users\diego\OBSIDIAN\` (sí visible desde WSL2 en `/mnt/c/Users/diego/OBSIDIAN/`, sin la penalización de rendimiento de rutas `/mnt/g/` ni las capas extra de sincronización de Drive).
- Reapuntado Obsidian a la nueva ruta (ahora reconoce dos vaults independientes: `demian` y `bitacora_aztlan`, cada uno con su propio `.obsidian/`).
- Reapuntado GitHub Desktop con "Locate..." a la nueva ubicación del repo — sincronización confirmada funcionando.

**Estructura resultante:**
```
C:\Users\diego\OBSIDIAN\
  demian\              ← vault del proyecto (memoria del agente + docs del repo)
  bitacora_aztlan\      ← vault del diario personal (NO tocado por el agente)
```

### 14. Skill de Obsidian instalado
- Se encontraron 4 skills con el slug "obsidian" en ClawHub — se eligió **`@steipete/obsidian`** por ser del propio creador de OpenClaw (Peter Steinberger), la fuente más confiable de las 4.
- `clawhub install` (sin `npx`) dio `command not found` — mismo patrón de PATH que con `openclaw` al inicio. Solución: usar `npx clawhub@latest install @steipete/obsidian`.
- **Hallazgo importante**: instalar un "skill" vía ClawHub no lo registra como "plugin" de OpenClaw con su propio campo de configuración (`openclaw config set plugins.entries.obsidian...` dio warning `plugin not found: obsidian (stale config entry ignored)`). Un skill funciona distinto: es un conjunto de instrucciones que el agente usa cuando se le da el contexto directamente en la conversación (ej. la ruta del vault), no algo que se configure una sola vez de forma persistente vía JSON.

### 🎉 Hito: memoria de Obsidian funcionando de extremo a extremo (corregido)

**Primera prueba (invalidada):** Se le pidió al agente escribir una nota de prueba, y "funcionó" — pero se detectó después que el comando `mkdir -p ... && echo ...` ya lo había corrido Diego manualmente en la terminal de Ubuntu en un paso anterior de la sesión. La prueba no demostraba que el agente ejecutara nada por su cuenta.

**Segunda prueba (también falló, pero reveló la causa real):** Se borró el archivo y se le pidió al agente crear una nota nueva vía Telegram (sin tocar la terminal). El agente respondió repetidamente "ya la creé" en loop, pero el archivo **no existía en disco** — estaba alucinando la ejecución. Causa raíz encontrada:
```
"profile": "messaging"
```
Este perfil (configurado días atrás para contener a `qwen2.5:3b`, que se enredaba con herramientas) **no incluye herramientas de sistema de archivos/shell** — solo permite conversación. El agente no tenía forma real de escribir el archivo, y en vez de decir "no puedo", alucinó que lo había hecho.

**Solución:**
```bash
openclaw config set tools.profile coding
systemctl --user stop openclaw-gateway
sleep 3
systemctl --user start openclaw-gateway
```
Con `tools.profile: coding` + Gemini (que sí maneja herramientas con criterio, a diferencia de los modelos locales), la tercera prueba fue exitosa y verificada de forma inequívoca: nota creada desde Telegram, confirmada en el Explorador de Windows sin usar la terminal.

**Lección aprendida:** Cuando un agente dice "ya lo hice" pero el resultado no aparece, verificar dos cosas antes de asumir un bug del skill: (1) si el perfil de herramientas activo (`tools.profile`) realmente incluye la categoría de herramienta necesaria, y (2) si la prueba está genuinamente aislada de cualquier acción manual previa del propio usuario en la misma sesión de trabajo.

### Nota pendiente para uso continuo
Como el skill no tiene una ruta de vault "recordada" de forma persistente en la configuración, cada vez que se le pida al agente trabajar con el vault, hay que **darle la ruta explícitamente** (o considerar agregarla al `USER.md` como contexto permanente, ya que ese sí se lee siempre) — pendiente de decidir cuál enfoque conviene más.

---

## ✅ Completado (continuación — 4 de agosto)

### 15. Limpieza de pendientes de la sesión anterior
- **Ollama detenido permanentemente**: `sudo systemctl disable ollama && sudo systemctl stop ollama` → confirmado `inactive (dead)`. Ya no aparece la advertencia de "WSL2 crash-loop risk" en el arranque del Gateway.
- **Ruta del vault agregada a `USER.md`** de forma persistente:
  ```
  - Vault de Obsidian del proyecto Demian: /mnt/c/Users/diego/OBSIDIAN/demian
  ```
- **Limpieza de `openclaw.json`**:
  - Bloque `agents.defaults.models` reducido de 9 entradas (con todos los intentos fallidos de OpenRouter/Ollama) a solo 2: `google/gemini-2.5-flash` (activo) y `ollama/qwen2.5:7b` (respaldo offline futuro).
  - Entrada obsoleta `plugins.entries.obsidian` eliminada (el skill de ClawHub no se registra como plugin con config propia — ya lo sabíamos, esta entrada era vestigial).
  - **Cuidado importante al editar JSON a mano en `nano`**: fácil dejar una coma de más o de menos al borrar/mover entradas. Flujo seguro usado: editar → `openclaw config validate` (sin reiniciar nada) → solo si valida OK, recién ahí `systemctl stop/start`. Este flujo evitó dejar el Gateway caído por un JSON inválido.
  - Confirmado tras reinicio: `Config valid`, sin warnings, `agent model: google/gemini-2.5-flash` cargando correctamente.

---

## 🔄 En progreso

### Pendiente:
- [ ] Configurar Cloudflare Tunnel + resolver `trusted_proxies_missing`
- [ ] Evaluar laptop (Ryzen 7 5700U, gráficos integrados — no resuelve la necesidad de GPU dedicada) como posible migración futura
- [ ] Decidir si se integra OpenCode para tareas de programación

---

## Lecciones aprendidas (para no repetir)

1. **Nunca escribir un ID de modelo "a ojo"** sin el prefijo del proveedor (`openrouter/...`) — causa el error de "Unknown model" registrado bajo el proveedor equivocado.
2. **El catálogo de modelos gratis de OpenRouter cambia constantemente** — un ID que funcionaba hace días puede dar `model not found`. Verificar en `openrouter.ai/models` antes de asumir.
3. **OAuth entre Windows y WSL2 tiene fricción conocida** con el callback local — la solución manual (pegar la URL completa) es confiable, mejor recordar este paso que perder tiempo reintentando el automático.
4. **No todos los modelos "gratis" son buenos para tool calling** — algunos están pensados solo para chat conversacional y fallan al usar herramientas del agente. Priorizar los que se anuncian explícitamente para uso agéntico.
5. **`sudo apt update && upgrade` en WSL2 recién instalado casi siempre requiere habilitar systemd después** — no es un error real, es un paso esperado del proceso.
