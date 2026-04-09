# Delta Chat Bots REST API Connector - Render Deployment

Esta guía explica cómo desplegar el Delta Chat Bots REST API Connector en [Render](https://render.com).

## Requisitos Previos

1. Una cuenta en Render (gratuita)
2. Tu repositorio en GitHub/GitLab
3. Un bot de Delta Chat configurado (o créalo después del deploy)

## Pasos para Desplegar

### Opción 1: Deploy Automático desde GitHub/GitLab

1. **Prepara tu repositorio:**
   - Asegúrate de que este repositorio esté en GitHub o GitLab
   - Verifica que los archivos `requirements.txt`, `Procfile` y `render.yaml` estén presentes

2. **Crea un nuevo Web Service en Render:**
   - Ve a https://render.com y loguéate
   - Click en "New +" → "Web Service"
   - Conecta tu repositorio de GitHub/GitLab

3. **Configura el servicio:**

   ```
   Name: delta-chat-bot-api
   Region: Frankfurt (eu-central) o el más cercano a ti
   Branch: main (o tu rama principal)
   Root Directory: (déjalo vacío si está en la raíz)
   Runtime: Python 3
   Build Command: pip install -r requirements.txt
   Start Command: python main.py serve
   ```

4. **Configura las Variables de Entorno:**

   En la sección "Environment", añade:

   ```
   BOT_CLI_NAME=RenderBot
   LOG_LEVEL=info
   
   ENABLE_WEBHOOK=false
   # WEBHOOK_URL=https://tu-webhook.com/endpoint
   # WEBHOOK_AUTH_TOKEN=tu-token-secreto
   
   API_HOST=0.0.0.0
   API_PORT=10000
   API_KEY=cambia-esta-clave-por-una-muy-segura
   
   MEDIA_DIR=/tmp/DeltaChatBotsRestAPI_media
   ```

5. **Selecciona el Plan:**
   - **Free**: $0/mes (se duerme después de 15 min de inactividad)
   - **Starter**: $7/mes (siempre activo, recomendado para producción)

6. **Click en "Create Web Service"**

### Opción 2: Usar render.yaml (Infrastructure as Code)

El archivo `render.yaml` incluido define automáticamente la configuración del servicio.

1. Sube tu código a GitHub/GitLab
2. En Render, click en "New +" → "Blueprint"
3. Conecta tu repositorio
4. Render leerá automáticamente el `render.yaml` y configurará todo

## Configuración del Bot Después del Deploy

Una vez desplegado, necesitas inicializar el bot:

### Método 1: Usando la Consola de Render

1. Ve a tu servicio en el dashboard de Render
2. Click en "Shell" (disponible en planes pagos)
3. Ejecuta:

```bash
python main.py init DCACCOUNT:https://nine.testrun.org/new
python main.py config displayname "Mi Bot en Render"
python main.py link
```

### Método 2: Usando la API RPC

Si ya tienes un bot configurado localmente, puedes copiar la carpeta de configuración:

1. Localmente, encuentra la carpeta del bot (usualmente en `~/.local/share/deltabot-cli/REST_API_Relay/`)
2. Copia los archivos de configuración
3. En Render, usa la consola o monta un volumen persistente

### Método 3: Script de Inicialización Automática

Puedes crear un script que verifique si el bot está inicializado:

```python
#!/usr/bin/env python3
# init_bot.py
import os
import subprocess
import sys

def is_bot_initialized():
    """Verifica si el bot ya está configurado"""
    result = subprocess.run(
        ["python", "main.py", "list"],
        capture_output=True,
        text=True
    )
    return "Account" in result.stdout

if __name__ == "__main__":
    if not is_bot_initialized():
        print("Bot no inicializado. Iniciando...")
        # Aquí podrías añadir lógica para inicializar automáticamente
        # pero necesitarías las credenciales de forma segura
        print("Por favor inicializa el bot manualmente usando la consola de Render")
        sys.exit(1)
    else:
        print("Bot ya está inicializado")
        sys.exit(0)
```

## URL Pública de la API

Una vez desplegado, Render te dará una URL como:
```
https://delta-chat-bot-api.onrender.com
```

Tu API estará disponible en:
```
https://delta-chat-bot-api.onrender.com/health
https://delta-chat-bot-api.onrender.com/rpc
https://delta-chat-bot-api.onrender.com/media
```

## Ejemplos de Uso

### Health Check
```bash
curl https://delta-chat-bot-api.onrender.com/health \
  -H "Authorization: Bearer tu-api-key-segura"
```

### Enviar Mensaje
```bash
curl https://delta-chat-bot-api.onrender.com/rpc \
  -X POST \
  -H "Authorization: Bearer tu-api-key-segura" \
  -H "Content-Type: application/json" \
  -d '{
    "method": "send_msg",
    "params": [1, 10, {"text": "Hola desde Render!"}]
  }'
```

## Consideraciones Importantes

### 1. Persistencia de Datos
- **Plan Gratuito**: Los datos se pierden cuando el servicio se reinicia
- **Plan Pago**: Usa "Persistent Disk" para guardar la configuración del bot

Para añadir almacenamiento persistente:
```yaml
# En render.yaml
disk:
  name: bot-data
  mountPath: /root/.local/share/deltabot-cli
  sizeGB: 1
```

### 2. Timeout y Sleep
- **Free tier**: El servicio se duerme después de 15 minutos de inactividad
- La primera request después de dormir tarda ~30 segundos en responder
- **Solución**: Usa un servicio de uptime monitoring (ej: UptimeRobot) para hacer ping cada 10 minutos

### 3. Logs y Debugging
- View logs en tiempo real desde el dashboard de Render
- Configura `LOG_LEVEL=debug` para más detalle
- Los logs se retienen por 7 días en free tier, 30 días en paid

### 4. Seguridad
- ✅ Usa HTTPS automáticamente (Render lo provee)
- ✅ API Key requerida para todos los endpoints
- ⚠️ Nunca commits `.env` con secrets al repositorio
- ✅ Usa Environment Variables de Render para secrets

### 5. Límites del Plan Gratuito
- 750 horas/mes (suficiente para 1 servicio siempre activo)
- 512 MB RAM
- CPU compartida
- Sin almacenamiento persistente incluido

## Troubleshooting

### Error: "ModuleNotFoundError: No module named 'deltabot_cli'"
**Solución**: Asegúrate de que `requirements.txt` esté correcto y el build command sea `pip install -r requirements.txt`

### Error: "Address already in use"
**Solución**: Render asigna el puerto automáticamente via variable `PORT`. Modifica `config.py`:

```python
# En config.py, cambia:
API_PORT = os.environ.get("PORT", os.environ.get("API_PORT", "8000"))
API_HOST = os.environ.get("API_HOST", "0.0.0.0")
```

### El servicio no responde después de deploy
**Solución**: 
1. Revisa los logs en el dashboard
2. Verifica que el bot esté inicializado
3. Asegúrate de que `API_HOST=0.0.0.0`

### Webhooks no funcionan
**Solución**:
1. Verifica que `ENABLE_WEBHOOK=true`
2. Asegúrate de que `WEBHOOK_URL` sea una URL pública accesible
3. Revisa los logs para ver si hay errores de conexión

## Actualizar el Deploy

Para actualizar tu servicio:

1. Haz push de cambios a tu rama principal
2. Render detectará automáticamente los cambios y redeployará
3. Puedes ver el progreso en el dashboard

O manualmente:
1. Ve al dashboard de Render
2. Click en "Manual Deploy"
3. Selecciona la rama y click "Deploy"

## Monitoreo

Render provee métricas básicas:
- CPU usage
- Memory usage
- Request count
- Response times

Para monitoreo avanzado, integra con:
- Sentry (error tracking)
- Datadog (métricas avanzadas)
- UptimeRobot (uptime monitoring)

## Costos Estimados

| Componente | Free Tier | Starter |
|------------|-----------|---------|
| Web Service | $0/mes | $7/mes |
| Persistent Disk (1GB) | N/A | $0.50/mes |
| **Total** | **$0/mes** | **$7.50/mes** |

## Recursos Adicionales

- [Documentación de Render](https://render.com/docs)
- [Python en Render](https://render.com/docs/deploy-fastapi)
- [Variables de Entorno](https://render.com/docs/environment-variables)
- [Persistent Disks](https://render.com/docs/disks)

---

**Nota**: Esta configuración está optimizada para Render. Para otros proveedores (Heroku, Railway, Fly.io), los principios son similares pero los detalles pueden variar.
