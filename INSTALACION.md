# Instalación — Demian

Guía paso a paso para levantar el proyecto en Endor PC (Windows 11 + WSL2).

## Requisitos previos

- Windows 11 con virtualización habilitada en BIOS.
- Cuenta de Cloudflare con un dominio propio (`claw.tu-dominio.example`).
- Bot de Telegram creado con @BotFather (guardar el token).
- API key de Anthropic. Opcional: acceso a Gemini Plus como proveedor alternativo.
- Cuenta de GitHub con un token de acceso de scope limitado (solo los repos de este proyecto).

## 1. WSL2 y Ubuntu

```powershell
wsl --install
```
Reinicia. Al volver, Ubuntu pedirá crear un usuario y contraseña Linux (distinto de tu login de Windows).

Verificar versión:
```powershell
wsl -l -v
```
Debe mostrar `VERSION 2` para Ubuntu. Si dice 1:
```powershell
wsl --set-version Ubuntu 2
```

## 2. Instalar OpenClaw

Dentro de la terminal de Ubuntu:
```bash
curl -fsSL https://openclaw.ai/install.sh | bash
openclaw onboard --install-daemon
```
El flag `--install-daemon` registra el gateway como servicio systemd de usuario, para que persista tras reinicios.

Si `openclaw` no se reconoce como comando:
```bash
echo 'export PATH="$(npm prefix -g)/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

Verificar que el servicio está activo:
```bash
systemctl --user status openclaw-gateway
openclaw gateway status
```
**Confirmar el puerto real** que reporta `openclaw gateway status` — no asumir uno fijo sin verificarlo primero.

## 3. Conectar Telegram

Durante el wizard de `openclaw onboard`, ingresar el `TELEGRAM_BOT_TOKEN`. Completar el proceso de "hatch" (emparejamiento). Configurar el allowlist de chat ID para que el bot solo responda a tu cuenta.

Probar enviando un mensaje al bot.

## 4. Vault de Obsidian

- Instalar Obsidian en Windows (no en WSL2).
- Crear o ubicar el vault en, por ejemplo: `C:\Users\tu_usuario\ObsidianVault`.
- Estructura del vault:
  ```
  ObsidianVault/
    diario/
    universidad/
    proyectos/
    agent-memory/
  ```
- Inicializar como repo git dentro de esa carpeta (puede hacerse desde WSL2 accediendo a `/mnt/c/Users/tu_usuario/ObsidianVault`).
- Instalar el skill de OpenClaw:
  ```bash
  npx clawhub@latest install obsidian
  ```
- Configurar la ruta en `openclaw.json` o variable de entorno:
  ```
  OBSIDIAN_VAULT_PATH=/mnt/c/Users/tu_usuario/ObsidianVault
  ```

> Nota de sincronización: al escribir desde WSL2 hacia `/mnt/c/`, el rendimiento de I/O es más lento que un filesystem nativo, pero suficiente para archivos de texto. Evitar que el agente escriba en un archivo mientras Obsidian lo tiene abierto activamente para minimizar riesgo de conflicto. Empezar con commits/push manuales antes de automatizarlos.

## 5. Cloudflare Tunnel

```bash
cloudflared tunnel login
cloudflared tunnel create demian
```

Archivo `~/.cloudflared/config.yml`:
```yaml
tunnel: <TUNNEL_ID>
credentials-file: /home/tu_usuario/.cloudflared/<TUNNEL_ID>.json
ingress:
  - hostname: claw.tu-dominio.example
    service: http://localhost:<PUERTO_CONFIRMADO_EN_PASO_2>
  - service: http_status:404
```

```bash
cloudflared tunnel route dns demian claw.tu-dominio.example
sudo cloudflared service install
sudo systemctl enable --now cloudflared
```

Después:
- Activar **Cloudflare Access** delante del hostname (login obligatorio).
- Configurar **reglas de rate limiting** en el panel de Cloudflare (WAF → Rate Limiting Rules) para el hostname.
- Confirmar que el puerto del gateway **no** está expuesto directamente en el router — solo accesible vía el tunnel.

## 6. Arranque automático tras reinicio/corte de luz

- En BIOS: habilitar "Restore on AC Power Loss" para que Endor PC encienda sola tras un corte eléctrico.
- En Windows: configurar el arranque de WSL2 y del daemon de OpenClaw al iniciar sesión (Task Scheduler apuntando a un script que ejecute `wsl` y verifique el servicio).

## 7. Sub-agentes iniciales

Empezar solo con:
1. Agente Universidad/Código
2. Agente Diario

Ver `docs/ARQUITECTURA.md` para su configuración y `docs/PLAN-DE-ACCION.md` para cuándo expandir a DevOps y Web.

## 8. Verificación final

- [ ] Mensaje a Telegram → respuesta del agente.
- [ ] `https://claw.tu-dominio.example` pide login de Cloudflare Access antes de mostrar el panel.
- [ ] Cambios manuales en el vault de Obsidian se reflejan en la siguiente interacción del agente.
- [ ] `systemctl --user status openclaw-gateway` y `systemctl status cloudflared` ambos activos tras un reinicio de Windows.
