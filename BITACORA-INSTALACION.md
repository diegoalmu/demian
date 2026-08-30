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

## ⚠️ Nota sobre continuidad entre sesiones de trabajo

Debido a límites de tokens en conversaciones largas con Claude, el trabajo de este proyecto a veces continúa en conversaciones nuevas que no tienen acceso al historial de la anterior. Esta bitácora es la memoria compartida entre esas sesiones — **mantenerla actualizada después de cada cambio importante es crítico**, no opcional.

## ✅ Completado (continuación — 30 de agosto)

### 16. Migración de sistema operativo: Windows 11 + WSL2 → Linux Mint 22.3 nativo
Entre la sesión del 5 de agosto y esta (30 de agosto), se realizó una migración completa de Endor PC de Windows 11 con WSL2 a **Linux Mint 22.3 nativo** (Windows eliminado por completo). Esta migración se hizo en una sesión de trabajo distinta cuyo historial no está disponible — se reconstruyó el estado por verificación directa en terminal, no por continuidad de contexto.

**Beneficio esperado de la migración:** elimina de raíz el problema de rendimiento de `/mnt/c/` documentado en sesiones anteriores, y simplifica la arquitectura al quitar la capa WSL2 completa.

**Estado verificado tras la migración (30 de agosto):**
- ✅ Vault de Obsidian sobrevivió: ahora en `/home/endoralex/OBSIDIAN/demian` (ruta nativa, ya sin prefijo `/mnt/c/`)
- ✅ `~/.openclaw/workspace/IDENTITY.md` y `USER.md` sobrevivieron intactos (nombre "Demian", idioma español, ruta de vault actualizada)
- ✅ Modelo sigue configurado correctamente: `google/gemini-2.5-flash`
- ⚠️ **Encontrado y corregido**: el archivo de servicio `~/.config/systemd/user/openclaw-gateway.service` existía (creado 12 de agosto, durante la migración) pero **no estaba activado** — el Gateway corría como proceso suelto lanzado manualmente (`ps aux` lo mostró sin supervisión de systemd). Esto significaba que un reinicio de la PC o cerrar la terminal habría tumbado a Demian sin que se levantara solo.
  - Solución: `kill <PID_del_proceso_suelto>` → `systemctl --user daemon-reload && systemctl --user enable openclaw-gateway && systemctl --user start openclaw-gateway` → confirmado `active (running)` como servicio real.
- ❓ **Sin confirmar / probablemente nunca implementado**: se mencionó en esta sesión una supuesta configuración inicial de "bloqueo total de red (nada entra, nada sale)" como medida de seguridad de base. Verificado con `sudo ufw status verbose` → **`ufw` está inactivo**, sin ninguna regla. No hay registro de esta configuración en ninguna sesión anterior documentada aquí. Conclusión: o se perdió por completo en la migración, o era una intención que nunca se llegó a implementar, o corresponde a una sesión con otra IA (Gemini/Kimi/Qwen/DeepSeek) que Diego evalúa en paralelo. **No se puede confirmar retroactivamente — se documenta como pendiente a definir desde cero, no como regresión de algo que sabemos que existió.**

### 17. Web search habilitado con Gemini como proveedor
- Causa del fallo original: `web_search` en OpenClaw usa Brave Search por defecto, que requiere API key propia no configurada — sin ella, la herramienta falla (a veces silenciosamente, con el agente alucinando que sí buscó).
- Solución aplicada: `openclaw config set tools.web.search.provider gemini` — reutiliza la cuenta de Gemini ya configurada, sin necesitar una key nueva de Brave.

### 18. Diagnóstico extenso: `web_search` sigue sin funcionar pese a key correcta

**Contexto**: se solicitó habilitar búsqueda web para uso escolar de los hijos de Diego (21, 17, 15 años) antes de avanzar con Cloudflare.

**Camino de diagnóstico (largo, documentado para no repetirlo):**

1. Configurado `tools.web.search.provider gemini` → seguía fallando con "GEMINI_API_KEY no configurada".
2. Se probó `plugins.entries.google.config.webSearch.apiKey` vía `config set` → se guardó en el JSON pero el error persistió idéntico.
3. Se investigó y confirmó (vía `docs.openclaw.ai/tools/web`) que la herramienta también acepta `GEMINI_API_KEY` como variable de entorno — probado vía `~/.openclaw/.env`, **no funcionó** porque el `.service` de systemd no tiene `EnvironmentFile=` apuntando ahí.
4. Se creó override correcto: `~/.config/systemd/user/openclaw-gateway.service.d/override.conf` con `Environment="GEMINI_API_KEY=..."` → confirmado con `systemctl --user cat` que se aplicó.
5. **Verificación definitiva de que la key sí llega completa al proceso real** (no solo a la config de systemd):
   ```bash
   cat /proc/$(pgrep -f "openclaw.*gateway")/environ | tr '\0' '\n' | grep GEMINI_API_KEY | wc -c
   ```
   → devolvió `55`, el largo exacto esperado (`GEMINI_API_KEY=` + 39 caracteres de key + salto de línea). **La key está bien configurada y llega al proceso.**
6. **Error grave encontrado y corregido en el camino**: al intentar `openclaw config set tools.allow '["group:web"]' --merge --json` para forzar el permiso de `web_search`, el flag `--merge` **no se comportó como merge aditivo** — convirtió `tools.allow` en una lista exclusiva, bloqueando 25+ herramientas que sí funcionaban (`exec`, `read`, `write`, `cron`, etc., incluyendo todo lo de Obsidian). **Revertido** con `openclaw config set tools.allow '[]' --json`. **Lección: no usar `tools.allow` para agregar una sola herramienta sin verificar antes qué contiene el perfil activo — puede convertirse en lista exclusiva y romper todo lo demás.**
7. Confirmado en logs (`journalctl`) que el modelo responde con `status=200` **sin que se registre ningún intento de llamar a `web_search`** — el modelo ni siquiera está invocando la herramienta, no es un fallo de la herramienta en sí.
8. Al pedirle al agente listar sus herramientas disponibles (para confirmar si `web_search` aparece registrada), la API de Gemini empezó a devolver: `"The AI service is temporarily overloaded. Please try again in a moment."` — error de saturación del lado de Google, no relacionado con la configuración. **Diagnóstico interrumpido por esta causa externa, pendiente de reintentar.**

**Estado al cierre de esta sesión**: sin resolver. La API key está técnicamente bien configurada y confirmada en el proceso, pero `web_search` no se ejecuta. Hipótesis principales a probar en la próxima sesión:
- El modelo simplemente no está decidiendo invocar la herramienta (posible tema de prompt/instrucción, no de configuración).
- La herramienta `web_search` podría no estar registrada como disponible en esta versión/instalación pese a la configuración de proveedor (pendiente confirmar con `agents_list` de herramientas o revisando plugins cargados en el log de arranque: `http server listening (N plugins: ...)`).

**Actualización del diagnóstico — causa raíz real encontrada**: el problema **no era `web_search` en absoluto** — era que **Gemini 2.5 Flash estaba saturado del lado de Google** (`503 UNAVAILABLE — high demand`) para *cualquier* solicitud, no solo para búsqueda. Confirmado porque:
- El error idéntico ocurría con mensajes que no requerían búsqueda en absoluto.
- Cambiar el proveedor de búsqueda a `duckduckgo` (sin API key, sin relación con Google) **no cambió el error** — porque el fallo ocurre antes de siquiera intentar buscar, en la generación de la respuesta del modelo en sí.
- Los logs muestran una progresión de `503` (servicio sobrecargado) a **`429`** (rate limit / cuota excedida) tras varios intentos seguidos en poco tiempo — probablemente agravado por la cantidad de pruebas hechas en esta sesión de diagnóstico.

**Conclusión**: no hay nada mal configurado. Es un problema temporal del lado de Google. **Acción tomada**: pausar todos los mensajes por al menos 10-15 minutos para dejar que se enfríe cualquier límite de tasa antes de reintentar.

**Próximo paso real**: cuando se retome, probar primero un mensaje simple sin pedir búsqueda (para confirmar que el modelo en general ya respondió), y solo después probar específicamente la búsqueda web.

### 19. Causa raíz real de todos los fallos: cuota gratuita de la API de Gemini agotada

Tras el diagnóstico extenso de `web_search` (punto 18), se confirmó la causa raíz real revisando logs con el error específico:
```
Google Generative AI API error (429): You exceeded your current quota, please check your plan and billing details. [code=RESOURCE_EXHAUSTED]
```

**Aclaración importante para el proyecto**: la suscripción de **Gemini Plus** (la app de consumo, tipo Google One) es un producto **separado** de la **API de Gemini para desarrolladores** (la que usa OpenClaw vía `generativelanguage.googleapis.com`). Tener Gemini Plus **no** otorga cuota ilimitada ni mayor en la API — esta tiene su propio tier gratuito limitado, independiente de la suscripción de consumo. Las docenas de peticiones hechas durante el diagnóstico de `web_search` agotaron esa cuota gratuita diaria.

**Decisión tomada**: no habilitar facturación en Google Cloud por ahora (evitar gasto no controlado durante fase de pruebas). Se opta por:
- Esperar el reset diario de la cuota gratuita.
- Mantener Ollama como respaldo local para cuando la cuota de Gemini esté agotada.
- **Precaución para el futuro**: evitar sesiones de diagnóstico con muchos reintentos seguidos contra la API de Gemini — cada intento (exitoso o fallido) consume cuota.

### 20. Ollama no sobrevivió la migración a Linux Mint — reinstalado desde cero
- Confirmado con `which ollama` (sin resultado) y `systemctl status ollama` (`Unit could not be found`): Ollama **no estaba instalado** en la instalación de Mint — no se puede reactivar un servicio que nunca se instaló aquí. Toda la instalación anterior de Ollama vivía en el sistema Windows+WSL2 ya eliminado.
- **Hardware confirmado en Mint nativo**: mismo procesador que siempre, **Intel i5-8400T** (6 núcleos, sin GPU dedicada) — la migración a Mint no cambió el hardware, solo quitó la capa de virtualización. RAM total ahora visible: **31GB** (vs. 15GB que WSL2 exponía), con ~23GB libres.
- Reinstalación limpia, sin el problema de `zstd` faltante que hubo en WSL2:
  ```bash
  curl -fsSL https://ollama.com/install.sh | sh
  ollama pull qwen2.5:7b
  ```
- Confirmado: "WARNING: No NVIDIA/AMD GPU detected. Ollama will run in CPU-only mode" — esperado, no hay GPU en este hardware.
- Mitigación de memoria aplicada igual que en la instalación anterior — **de nuevo** `sudo systemctl edit ollama` guardó el archivo **vacío** (mismo bug/comportamiento que con OpenClaw el 30 de julio). Solución igual que la vez pasada: crear el override manualmente:
  ```bash
  sudo mkdir -p /etc/systemd/system/ollama.service.d
  sudo nano /etc/systemd/system/ollama.service.d/override.conf
  # [Service]
  # Environment="OLLAMA_KEEP_ALIVE=5m"
  sudo systemctl daemon-reload
  sudo systemctl restart ollama
  ```
  **Nota para el futuro**: `systemctl edit` (tanto `--user` como con `sudo` para servicios de sistema) parece no guardar correctamente en este flujo de trabajo — ir directo al método manual de crear `override.conf` ahorra un ciclo de diagnóstico.

### 21. Diagnóstico de lentitud extrema al probar Ollama (5+ minutos sin respuesta)

Al probar `qwen2.5:7b` recién instalado, una consulta simple tardó más de 5 minutos con la CPU al máximo. Diagnóstico vía `journalctl -u ollama`:
```
truncating input prompt limit=2050 prompt=17191 keep=4 new=2050
...
prompt processing, n_tokens = 1536, progress = 0.75, t = 120.51 s / 12.75 tokens per second
```

**Causa real**: no es un bug — el contexto acumulado en la sesión (17,191 tokens, posiblemente arrastrando historial pese al `/new`) se truncó a 2050 tokens límite, y el hardware (i5-8400T, sin GPU) procesa esos tokens de entrada a ~13 tokens/segundo. A ese ritmo, cualquier prompt con contexto sustancial toma varios minutos solo para *procesar la entrada*, antes de generar la respuesta.

**Conclusión reforzada**: Ollama en este hardware es viable únicamente para mensajes cortos y sin historial acumulado (sesión recién limpiada con `/new`). Para cualquier conversación con contexto real, la espera es impráctica. Esto confirma, con datos concretos de rendimiento, la decisión ya tomada de usar Gemini como modelo principal y reservar Ollama estrictamente como respaldo de emergencia de corta duración.

### Lección aprendida (proceso de trabajo)
Cuando el trabajo se retoma después de un corte de contexto (tokens agotados, sesión nueva), **verificar el estado real en la terminal antes de asumir cualquier configuración previa** — no confiar en el recuerdo de "creo que configuramos X", ya que puede mezclarse entre sesiones distintas o con trabajo hecho con otras herramientas de IA. Esta bitácora existe precisamente para reducir ese riesgo; actualizarla en cada hito es lo que la hace útil.

---

## 🔄 En progreso

### Pendiente:
- [ ] Confirmar que Gemini volvió a responder tras reset de cuota; volver a poner `google/gemini-2.5-flash` como `agents.defaults.model.primary` (se dejó en `ollama/qwen2.5:7b` temporalmente el 30 de agosto)
- [ ] Resolver `web_search` de forma definitiva una vez que la cuota de Gemini esté disponible de nuevo (causa de fondo ya no es un misterio — era cuota agotada, no configuración)
- [ ] Rotar las 3 credenciales expuestas en chat el 30 de agosto: token del Gateway, API key de Gemini, token del bot de Telegram
- [ ] Confirmar plan de acceso para los hijos de Diego (21, 17, 15 años) — sub-agente separado con perfil de herramientas acotado, antes de exponer con Cloudflare
- [ ] Configurar Cloudflare Tunnel + resolver `trusted_proxies_missing`
- [ ] Evaluar laptop (Ryzen 7 5700U, gráficos integrados — no resuelve la necesidad de GPU dedicada) como posible migración futura
- [ ] Decidir si se integra OpenCode para tareas de programación
- [ ] Actualizar `INSTALACION.md` y `ARQUITECTURA.md` para reflejar Linux Mint nativo (actualmente documentan Windows+WSL2)

---

## Lecciones aprendidas (para no repetir)

1. **Nunca escribir un ID de modelo "a ojo"** sin el prefijo del proveedor (`openrouter/...`) — causa el error de "Unknown model" registrado bajo el proveedor equivocado.
2. **El catálogo de modelos gratis de OpenRouter cambia constantemente** — un ID que funcionaba hace días puede dar `model not found`. Verificar en `openrouter.ai/models` antes de asumir.
3. **OAuth entre Windows y WSL2 tiene fricción conocida** con el callback local — la solución manual (pegar la URL completa) es confiable, mejor recordar este paso que perder tiempo reintentando el automático.
4. **No todos los modelos "gratis" son buenos para tool calling** — algunos están pensados solo para chat conversacional y fallan al usar herramientas del agente. Priorizar los que se anuncian explícitamente para uso agéntico.
5. **`sudo apt update && upgrade` en WSL2 recién instalado casi siempre requiere habilitar systemd después** — no es un error real, es un paso esperado del proceso.
