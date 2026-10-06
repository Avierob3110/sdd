# Manual técnico: Despliegue de una pila de IA local en Docker sobre Ubuntu

**Proyecto:** Despliegue IA local basado en SDD  
**Versión del manual:** 1.0  
**Rol:** Administrador de sistemas  
**Sistema objetivo:** Ubuntu Server 24.04 LTS o 26.04 LTS, amd64  
**Arquitectura:** Docker Engine + Docker Compose + NVIDIA GPU  
**Directorio de trabajo:** `$HOME/proyecto`

> **Nota de arquitectura.** Este manual parte de la especificación entregada. Se han mantenido sus servicios, nombres y puertos externos siempre que es técnicamente viable. Cuando la implementación actual de un proyecto usa un puerto interno o mecanismo distinto del indicado originalmente, se documenta la diferencia en lugar de inventarla.

---

## 1. Objetivo y arquitectura

El objetivo es desplegar una infraestructura de IA local en Ubuntu Server mediante contenedores Docker, con NVIDIA GPU y comunicación interna mediante una red Docker llamada `red-ia`.

Servicios:

| Servicio | Contenedor | Host | Interno | Función |
|---|---|---:|---:|---|
| Ollama | `ollama` | 11434 | 11434 | LLM local/API |
| Open WebUI | `openwebui` | 3000 | 8080 | Interfaz web |
| Hermes Agent | `hermes-agent` | 8000 | 8642 | Agente/gateway |
| OpenCode | `opencode` | 8443 | 4096 | IDE/agente de código web |
| ComfyUI | `comfyui` | 8188 | 8188 | Generación de imágenes/vídeo |
| YOLO | `yolo` | 5000 | 5000 | API de visión/detección |
| SearXNG | `searxng` | 8080 | 8080 | Metabuscador |
| RAG | integrado | — | — | Recuperación aumentada |

La red interna permite que los contenedores se resuelvan por nombre:

```text
ollama:11434
openwebui:8080
hermes-agent:8642
opencode:4096
comfyui:8188
yolo:5000
searxng:8080
```

La especificación original indicaba `http://hermesagent:8000` y `http://opencode:8443`. En esta implementación se usa el nombre real del contenedor `hermes-agent` y los puertos internos reales de sus servidores actuales; los puertos externos permanecen en los valores del proyecto.

---

# 2. Prerrequisitos

## 2.1 Hardware

Mínimos funcionales:

- CPU x86_64.
- RAM suficiente para el modelo elegido.
- GPU NVIDIA compatible.
- Almacenamiento rápido y suficiente para modelos.
- Conexión a Internet para descargar imágenes Docker, modelos y actualizaciones.

Para una pila de IA real, el almacenamiento y la VRAM son especialmente importantes. No se debe interpretar una GPU RTX 3050/4060 como garantía de que cualquier modelo podrá ejecutarse: el tamaño del modelo y su cuantización determinan el consumo.

## 2.2 Comprobar Ubuntu

```bash
cat /etc/os-release
uname -m
```

Debe aparecer Ubuntu 24.04/26.04 y una arquitectura compatible.

Actualizar:

```bash
sudo apt update
sudo apt upgrade -y
```

Instalar utilidades:

```bash
sudo apt install -y ca-certificates curl gnupg git openssl
```

---

# 3. Instalación del controlador NVIDIA

Primero comprobar si el controlador ya funciona:

```bash
nvidia-smi
```

Si muestra la GPU, versión del driver y memoria, el controlador está operativo.

Si no existe `nvidia-smi`, instalar el controlador mediante los paquetes recomendados por Ubuntu para la GPU concreta.

Después reiniciar:

```bash
sudo reboot
```

Volver a comprobar:

```bash
nvidia-smi
```

> El controlador NVIDIA se instala en el **host**. CUDA no debe confundirse con el NVIDIA Container Toolkit: el primero proporciona el soporte de GPU del sistema/driver y el segundo permite que los contenedores accedan a ella.

---

# 4. Instalación de Docker Engine

Eliminar paquetes que puedan entrar en conflicto:

```bash
sudo apt remove -y docker.io docker-compose docker-compose-v2 docker-doc   docker-buildx docker-ce docker-ce-cli containerd runc podman-docker
```

Instalar la clave oficial:

```bash
sudo apt update
sudo apt install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL   https://download.docker.com/linux/ubuntu/gpg   -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Crear el repositorio:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Actualizar e instalar:

```bash
sudo apt update

sudo apt install -y   docker-ce   docker-ce-cli   containerd.io   docker-buildx-plugin   docker-compose-plugin
```

Comprobar:

```bash
sudo systemctl enable --now docker
sudo systemctl status docker
docker --version
docker compose version
```

Prueba:

```bash
sudo docker run --rm hello-world
```

Docker documenta actualmente Ubuntu 24.04 y 26.04 como versiones LTS soportadas para Docker Engine. citeturn0search1

## 4.1 Permitir Docker sin sudo

Añadir el usuario actual al grupo:

```bash
sudo usermod -aG docker "$USER"
```

Cerrar sesión y volver a entrar.

Comprobar:

```bash
docker ps
```

> Pertenecer al grupo `docker` equivale en la práctica a disponer de privilegios elevados sobre el host. No debe concederse a usuarios no confiables.

---

# 5. NVIDIA Container Toolkit

Instalar dependencias:

```bash
sudo apt-get update

sudo apt-get install -y --no-install-recommends   ca-certificates   curl   gnupg2
```

Añadir repositorio:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey   | sudo gpg --dearmor   -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L   https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list   | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g'   | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

Instalar:

```bash
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

Configurar Docker:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
```

Reiniciar Docker:

```bash
sudo systemctl restart docker
```

NVIDIA documenta `nvidia-ctk runtime configure --runtime=docker` como el mecanismo para configurar Docker para utilizar el runtime NVIDIA. citeturn0search0

## 5.1 Prueba de GPU dentro de Docker

```bash
docker run --rm --gpus all nvidia/cuda:12.6.2-base-ubuntu24.04 nvidia-smi
```

Si se muestra la GPU desde dentro del contenedor, el passthrough está operativo.

Para entornos nuevos también puede utilizarse CDI, por ejemplo:

```bash
nvidia-ctk cdi list
```

y:

```bash
docker run --rm   --device nvidia.com/gpu=all   nvidia/cuda:12.6.2-base-ubuntu24.04   nvidia-smi
```

---

# 6. Estructura del proyecto

Crear:

```bash
mkdir -p "$HOME/proyecto"
cd "$HOME/proyecto"

mkdir -p   configs/searxng   yolo   opencode   scripts
```

Árbol:

```text
$HOME/proyecto/
├── .env
├── .env.example
├── docker-ollama.yml
├── docker-openwebui.yml
├── docker-hermes-agent.yml
├── docker-opencode.yml
├── docker-comfyui.yml
├── docker-yolo.yml
├── docker-searxng.yml
├── docker-rag.yml
├── configs/
│   └── searxng/
│       └── settings.yml
├── yolo/
│   ├── Dockerfile
│   └── app.py
├── opencode/
└── scripts/
```

Crear el fichero de entorno:

```bash
cp .env.example .env
chmod 600 .env
```

Generar secretos:

```bash
openssl rand -hex 32
```

Sustituir los valores `CHANGE_ME...` del `.env`.

---

# 7. Red Docker

Crear la red una sola vez:

```bash
docker network create --driver bridge red-ia
```

Comprobar:

```bash
docker network inspect red-ia
```

Los Compose utilizan:

```yaml
networks:
  red-ia:
    external: true
```

Esto permite que cada fichero Compose independiente conecte su contenedor a la misma red.

---

# 8. Persistencia

La especificación usa `$HOME` como raíz de datos.

Crear:

```bash
mkdir -p   "$HOME/ollama"   "$HOME/openwebui"   "$HOME/hermes"   "$HOME/opencode/workspace"   "$HOME/comfyui/models"   "$HOME/comfyui/custom-nodes"   "$HOME/comfyui/output"   "$HOME/yolo/models"   "$HOME/yolo/data"   "$HOME/searxng"
```

Comprobar:

```bash
ls -la "$HOME"/{ollama,openwebui,hermes,opencode,comfyui,yolo,searxng}
```

---

# 9. Ollama

## 9.1 Función

Ollama proporciona la API de modelos locales:

```text
http://ollama:11434
```

Desde el host:

```text
http://IP_DEL_SERVIDOR:11434
```

## 9.2 Fichero

`docker-ollama.yml`:

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "${OLLAMA_PORT:-11434}:11434"
    environment:
      - TZ=${TZ:-Europe/Madrid}
    volumes:
      - ${HOME}/ollama:/root/.ollama
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

networks:
  red-ia:
    external: true
```

Arrancar:

```bash
docker compose -f docker-ollama.yml up -d
```

Comprobar:

```bash
docker ps
docker logs ollama
```

API:

```bash
curl http://localhost:11434/api/tags
```

## 9.3 Descargar un modelo

Ejemplo:

```bash
docker exec -it ollama ollama pull llama3.2
```

Listar:

```bash
docker exec ollama ollama list
```

Probar:

```bash
docker exec -it ollama ollama run llama3.2
```

---

# 10. Open WebUI

Open WebUI puede utilizar Ollama como proveedor y mantiene sus datos en `/app/backend/data`. La documentación actual recomienda fijar `WEBUI_SECRET_KEY` para que las recreaciones del contenedor no invaliden las sesiones. citeturn0search3turn0search6

## 10.1 Compose

```yaml
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:cuda
    container_name: openwebui
    restart: unless-stopped
    ports:
      - "${OPENWEBUI_PORT:-3000}:8080"
    environment:
      - TZ=${TZ:-Europe/Madrid}
      - OLLAMA_BASE_URL=http://ollama:11434
      - WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY}
      - RAG_EMBEDDING_ENGINE=ollama
      - RAG_EMBEDDING_MODEL=nomic-embed-text
    volumes:
      - ${HOME}/openwebui:/app/backend/data
    depends_on:
      - ollama
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

networks:
  red-ia:
    external: true
```

Arrancar:

```bash
docker compose -f docker-openwebui.yml up -d
```

Acceder:

```text
http://IP_DEL_SERVIDOR:3000
```

## 10.2 Comprobar conexión con Ollama

Desde Open WebUI, comprobar la conexión al endpoint:

```text
http://ollama:11434
```

Desde el contenedor:

```bash
docker exec openwebui curl -s http://ollama:11434/api/tags
```

Si devuelve JSON con modelos, la comunicación funciona.

---

# 11. Hermes Agent

La implementación actual de Hermes Agent dispone de imagen Docker `nousresearch/hermes-agent` y utiliza `/opt/data` como directorio persistente. Su gateway expone actualmente el puerto 8642; por ello este proyecto publica el puerto 8000 del host hacia `8642` del contenedor. citeturn2search0turn2search6

## 11.1 Compose

```yaml
services:
  hermes-agent:
    image: nousresearch/hermes-agent:latest
    container_name: hermes-agent
    restart: unless-stopped
    command: gateway run
    ports:
      - "${HERMES_PORT:-8000}:8642"
    environment:
      - TZ=${TZ:-Europe/Madrid}
      - API_SERVER_ENABLED=true
      - API_SERVER_HOST=0.0.0.0
      - API_SERVER_KEY=${HERMES_API_SERVER_KEY}
      - PUID=${PUID:-1000}
      - PGID=${PGID:-1000}
    volumes:
      - ${HOME}/hermes:/opt/data
    shm_size: "1gb"
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

Arrancar:

```bash
docker compose -f docker-hermes-agent.yml up -d
```

Logs:

```bash
docker logs -f hermes-agent
```

Comprobar:

```bash
docker ps
```

### Importante

Hermes puede necesitar una configuración inicial. La documentación oficial recomienda ejecutar primero el asistente `setup` con el directorio persistente montado. citeturn2search0

Ejemplo:

```bash
docker run -it --rm   -v "$HOME/hermes:/opt/data"   nousresearch/hermes-agent setup
```

Después volver a levantar el Compose.

---

# 12. OpenCode

OpenCode dispone actualmente de interfaz web y de servidor HTTP. El servidor web utiliza `4096` como puerto predeterminado y puede protegerse mediante `OPENCODE_SERVER_PASSWORD`. citeturn3search1turn3search7

La especificación original asignaba `8443` como puerto externo; se mantiene ese puerto, pero se publica hacia el puerto interno real `4096`.

## 12.1 Compose

```yaml
services:
  opencode:
    image: ghcr.io/anomalyco/opencode:latest
    container_name: opencode
    restart: unless-stopped
    command: web --hostname 0.0.0.0 --port 4096
    ports:
      - "${OPENCODE_PORT:-8443}:4096"
    environment:
      - TZ=${TZ:-Europe/Madrid}
      - OPENCODE_SERVER_PASSWORD=${OPENCODE_SERVER_PASSWORD}
      - OPENCODE_SERVER_USERNAME=${OPENCODE_SERVER_USERNAME:-opencode}
    volumes:
      - ${HOME}/opencode:/root/.local/share/opencode
      - ${HOME}/opencode/workspace:/workspace
    working_dir: /workspace
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

Arrancar:

```bash
docker compose -f docker-opencode.yml up -d
```

Acceder:

```text
http://IP_DEL_SERVIDOR:8443
```

## 12.2 Conectar OpenCode con Ollama

OpenCode admite proveedores y configuraciones de modelos locales. Para este proyecto, el endpoint de Ollama es:

```text
http://ollama:11434
```

La configuración concreta de proveedor/modelo depende de la versión de OpenCode instalada. No se debe copiar una configuración antigua sin comprobar la sintaxis de la versión instalada.

Comprobar versión:

```bash
docker exec opencode opencode --version
```

---

# 13. ComfyUI

ComfyUI necesita persistencia especialmente para modelos, nodos personalizados y resultados.

El ejemplo utiliza una imagen Docker de ComfyUI con soporte NVIDIA y los directorios:

```text
models
custom_nodes
output
```

La imagen documenta el uso con `--gpus all` y estos montajes persistentes. citeturn4search1

## 13.1 Compose

```yaml
services:
  comfyui:
    image: ghcr.io/lecode-official/comfyui-docker:latest
    container_name: comfyui
    restart: unless-stopped
    ports:
      - "${COMFYUI_PORT:-8188}:8188"
    environment:
      - USER_ID=${PUID:-1000}
      - GROUP_ID=${PGID:-1000}
      - TZ=${TZ:-Europe/Madrid}
    volumes:
      - ${HOME}/comfyui/models:/opt/comfyui/models
      - ${HOME}/comfyui/custom-nodes:/opt/comfyui/custom_nodes
      - ${HOME}/comfyui/output:/opt/comfyui/output
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

networks:
  red-ia:
    external: true
```

Arrancar:

```bash
docker compose -f docker-comfyui.yml up -d
```

Acceder:

```text
http://IP_DEL_SERVIDOR:8188
```

Logs:

```bash
docker logs -f comfyui
```

---

# 14. YOLO

La especificación define YOLO como una API de visión artificial en el puerto 5000. Ultralytics proporciona imágenes Docker oficiales y actualmente documenta el acceso GPU mediante NVIDIA/CDI o `--gpus all`. citeturn1search0turn1search2

Como la especificación no define una imagen de servidor HTTP concreta para YOLO, se crea una API mínima con FastAPI sobre Ultralytics.

## 14.1 Dockerfile

```dockerfile
FROM ultralytics/ultralytics:latest

WORKDIR /app

COPY app.py /app/app.py

RUN pip install --no-cache-dir fastapi uvicorn python-multipart

EXPOSE 5000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "5000"]
```

## 14.2 API

```python
import os
from pathlib import Path

from fastapi import FastAPI, UploadFile, File
from fastapi.responses import JSONResponse
from ultralytics import YOLO

app = FastAPI(title="YOLO API", version="1.0")

MODEL_NAME = os.getenv("YOLO_MODEL", "yolo26n.pt")
MODEL_PATH = Path("/models") / MODEL_NAME

model = YOLO(str(MODEL_PATH) if MODEL_PATH.exists() else MODEL_NAME)


@app.get("/health")
def health():
    return {
        "status": "ok",
        "model": MODEL_NAME
    }


@app.post("/predict")
async def predict(file: UploadFile = File(...)):
    suffix = Path(file.filename or "image.jpg").suffix or ".jpg"
    tmp = Path("/tmp/input" + suffix)

    tmp.write_bytes(await file.read())

    try:
        results = model.predict(
            source=str(tmp),
            verbose=False
        )

        data = []

        for result in results:
            for box in result.boxes:
                data.append({
                    "class_id": int(box.cls[0]),
                    "confidence": float(box.conf[0]),
                    "xyxy": [
                        float(x)
                        for x in box.xyxy[0].tolist()
                    ]
                })

        return JSONResponse({
            "model": MODEL_NAME,
            "detections": data
        })

    finally:
        tmp.unlink(missing_ok=True)
```

## 14.3 Compose

```yaml
services:
  yolo:
    build:
      context: ./yolo
    container_name: yolo
    restart: unless-stopped
    ports:
      - "${YOLO_PORT:-5000}:5000"
    environment:
      - YOLO_MODEL=${YOLO_MODEL:-yolo26n.pt}
      - TZ=${TZ:-Europe/Madrid}
    volumes:
      - ${HOME}/yolo:/data
      - ${HOME}/yolo/models:/models
    networks:
      - red-ia
    ipc: host
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

networks:
  red-ia:
    external: true
```

Construir:

```bash
docker compose -f docker-yolo.yml build
```

Arrancar:

```bash
docker compose -f docker-yolo.yml up -d
```

Health check:

```bash
curl http://localhost:5000/health
```

Respuesta esperada:

```json
{
  "status": "ok",
  "model": "yolo26n.pt"
}
```

---

# 15. SearXNG

SearXNG proporciona búsquedas agregadas. Su documentación recomienda Compose y una configuración mediante `settings.yml`. citeturn1search6

## 15.1 Configuración

`configs/searxng/settings.yml`:

```yaml
use_default_settings: true

server:
  secret_key: "CAMBIAR"

  limiter: false

  image_proxy: false

search:
  formats:
    - html
    - json
```

Generar secreto:

```bash
openssl rand -hex 32
```

Pegar el valor en `secret_key`.

## 15.2 Compose

```yaml
services:
  searxng:
    image: searxng/searxng:latest
    container_name: searxng
    restart: unless-stopped
    ports:
      - "${SEARXNG_PORT:-8080}:8080"
    environment:
      - SEARXNG_BASE_URL=http://localhost:${SEARXNG_PORT:-8080}/
      - SEARXNG_SECRET=${SEARXNG_SECRET}
    volumes:
      - ./configs/searxng:/etc/searxng
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

Arrancar:

```bash
docker compose -f docker-searxng.yml up -d
```

Comprobar:

```bash
curl http://localhost:8080
```

API JSON:

```bash
curl 'http://localhost:8080/search?q=ubuntu&format=json'
```

---

# 16. RAG

## 16.1 Concepto

RAG significa Retrieval-Augmented Generation.

Flujo:

```text
Documentos
   |
   v
Fragmentación
   |
   v
Embeddings
   |
   v
Base/vector store
   |
   v
Consulta del usuario
   |
   v
Recuperación de fragmentos
   |
   v
Ollama / LLM
   |
   v
Respuesta
```

En esta arquitectura **RAG no necesita un contenedor independiente**. La propia especificación indica que está integrado con otros servicios.

Open WebUI puede utilizar Ollama para embeddings y almacenamiento de conocimiento.

## 16.2 Modelo de embeddings

Ejemplo:

```bash
docker exec ollama ollama pull nomic-embed-text
```

Comprobar:

```bash
docker exec ollama ollama list
```

## 16.3 Flujo de integración

```text
Usuario
   |
   v
Open WebUI :3000
   |
   +--------------------+
   |                    |
   v                    v
RAG / Knowledge       Ollama :11434
   |                    |
   |                    v
   |                 LLM
   |
   +----> embeddings
```

No se crea una falsa imagen Docker llamada `rag`. El fichero `docker-rag.yml` incluido en el proyecto documenta esta decisión.

---

# 17. Despliegue completo

## 17.1 Comprobar sintaxis

Desde:

```bash
cd "$HOME/proyecto"
```

Ejecutar:

```bash
docker compose --env-file .env -f docker-ollama.yml config
docker compose --env-file .env -f docker-openwebui.yml config
docker compose --env-file .env -f docker-hermes-agent.yml config
docker compose --env-file .env -f docker-opencode.yml config
docker compose --env-file .env -f docker-comfyui.yml config
docker compose --env-file .env -f docker-yolo.yml config
docker compose --env-file .env -f docker-searxng.yml config
```

Si no devuelve errores, la sintaxis Compose es correcta.

## 17.2 Arranque por fases

No es recomendable arrancar toda la IA a la vez en el primer despliegue.

### Fase 1: Ollama

```bash
docker compose --env-file .env -f docker-ollama.yml up -d
```

Comprobar:

```bash
docker ps
curl http://localhost:11434/api/tags
```

### Fase 2: Open WebUI

```bash
docker compose --env-file .env -f docker-openwebui.yml up -d
```

### Fase 3: SearXNG

```bash
docker compose --env-file .env -f docker-searxng.yml up -d
```

### Fase 4: ComfyUI

```bash
docker compose --env-file .env -f docker-comfyui.yml up -d
```

### Fase 5: YOLO

```bash
docker compose --env-file .env -f docker-yolo.yml up -d
```

### Fase 6: Hermes

```bash
docker compose --env-file .env -f docker-hermes-agent.yml up -d
```

### Fase 7: OpenCode

```bash
docker compose --env-file .env -f docker-opencode.yml up -d
```

---

# 18. Comprobación global

Listar:

```bash
docker ps --format "table {{.Names}}	{{.Status}}	{{.Ports}}"
```

Resultado esperado:

```text
ollama
openwebui
hermes-agent
opencode
comfyui
yolo
searxng
```

Red:

```bash
docker network inspect red-ia
```

GPU:

```bash
nvidia-smi
```

GPU desde Docker:

```bash
docker run --rm --gpus all   nvidia/cuda:12.6.2-base-ubuntu24.04   nvidia-smi
```

---

# 19. Pruebas de conectividad interna

Desde Ollama:

```bash
docker exec ollama getent hosts openwebui
docker exec ollama getent hosts searxng
docker exec ollama getent hosts opencode
```

Desde Open WebUI:

```bash
docker exec openwebui curl -s http://ollama:11434/api/tags
```

Desde YOLO:

```bash
docker exec yolo getent hosts ollama
```

Desde OpenCode:

```bash
docker exec opencode getent hosts ollama
```

Si `getent hosts` devuelve una IP de la red Docker, el DNS interno está funcionando.

---

# 20. Integración entre servicios

## 20.1 Open WebUI -> Ollama

Endpoint:

```text
http://ollama:11434
```

No utilizar:

```text
http://localhost:11434
```

desde Open WebUI, porque `localhost` dentro del contenedor apunta al propio contenedor.

Open WebUI documenta `OLLAMA_BASE_URL=http://ollama:11434` para una instalación Compose con Ollama. citeturn0search6

---

## 20.2 Open WebUI -> RAG

Open WebUI utiliza:

```text
Ollama
   |
   +--> modelo generativo
   |
   +--> modelo de embeddings
```

Ejemplo:

```bash
docker exec ollama ollama pull nomic-embed-text
```

Después configurar el conocimiento/RAG desde Open WebUI.

---

## 20.3 Hermes -> Ollama

La conexión debe utilizar:

```text
http://ollama:11434
```

No:

```text
http://localhost:11434
```

La configuración exacta de proveedor/modelo depende de la versión de Hermes.

---

## 20.4 OpenCode -> Ollama

Utilizar:

```text
http://ollama:11434
```

La versión actual de OpenCode admite proveedores configurables y servidores locales. La documentación oficial muestra también el uso de endpoints OpenAI-compatible para proveedores locales. citeturn3search0turn3search2

---

## 20.5 Hermes -> SearXNG

Endpoint:

```text
http://searxng:8080
```

El contenedor nunca debe usar:

```text
http://localhost:8080
```

porque eso apuntaría al propio Hermes.

---

## 20.6 OpenCode -> SearXNG

Si se implementa búsqueda externa mediante una herramienta/MCP compatible, el endpoint interno es:

```text
http://searxng:8080
```

La configuración concreta depende de la versión y del mecanismo de herramientas habilitado en OpenCode.

---

## 20.7 Hermes -> ComfyUI

Endpoint:

```text
http://comfyui:8188
```

---

## 20.8 Open WebUI -> ComfyUI

Endpoint:

```text
http://comfyui:8188
```

La integración exacta depende de las capacidades/extensiones habilitadas en la versión de Open WebUI.

---

## 20.9 Servicios -> YOLO

Endpoint:

```text
http://yolo:5000
```

Health:

```text
GET /health
```

Predicción:

```text
POST /predict
```

---

# 21. Prueba YOLO

Desde el host:

```bash
curl http://localhost:5000/health
```

Para una imagen:

```bash
curl   -X POST   -F "file=@imagen.jpg"   http://localhost:5000/predict
```

Respuesta aproximada:

```json
{
  "model": "yolo26n.pt",
  "detections": [
    {
      "class_id": 0,
      "confidence": 0.91,
      "xyxy": [10.2, 20.3, 300.4, 400.5]
    }
  ]
}
```

---

# 22. Gestión de logs

Todos los logs:

```bash
docker ps --format "{{.Names}}"
```

Servicio concreto:

```bash
docker logs ollama
docker logs openwebui
docker logs hermes-agent
docker logs opencode
docker logs comfyui
docker logs yolo
docker logs searxng
```

Seguimiento:

```bash
docker logs -f ollama
```

Últimas 100 líneas:

```bash
docker logs --tail 100 ollama
```

Con timestamps:

```bash
docker logs -t --tail 100 ollama
```

---

# 23. Estado y consumo

Estado:

```bash
docker ps
```

Todos los contenedores:

```bash
docker ps -a
```

Consumo:

```bash
docker stats
```

Solo uno:

```bash
docker stats ollama
```

GPU:

```bash
watch -n 1 nvidia-smi
```

---

# 24. Actualización

## 24.1 Actualizar un servicio

Ejemplo:

```bash
docker compose -f docker-ollama.yml pull
docker compose -f docker-ollama.yml up -d
```

Open WebUI:

```bash
docker compose -f docker-openwebui.yml pull
docker compose -f docker-openwebui.yml up -d
```

La documentación de Open WebUI recomienda `docker compose pull` seguido de `docker compose up -d`. citeturn0search6

SearXNG recomienda igualmente revisar las plantillas/configuración antes de actualizar y después ejecutar `pull` y `up -d`. citeturn1search6

---

# 25. Actualización de todos

```bash
for f in docker-*.yml; do
  case "$f" in
    docker-rag.yml) continue ;;
  esac

  docker compose --env-file .env -f "$f" pull
  docker compose --env-file .env -f "$f" up -d
done
```

Después:

```bash
docker ps
```

---

# 26. Backups

## 26.1 Crear directorio

```bash
mkdir -p "$HOME/backups"
```

## 26.2 Backup Open WebUI

```bash
tar -czf   "$HOME/backups/openwebui-$(date +%F).tar.gz"   "$HOME/openwebui"
```

## 26.3 Backup Hermes

```bash
tar -czf   "$HOME/backups/hermes-$(date +%F).tar.gz"   "$HOME/hermes"
```

## 26.4 Backup OpenCode

```bash
tar -czf   "$HOME/backups/opencode-$(date +%F).tar.gz"   "$HOME/opencode"
```

## 26.5 Backup SearXNG

```bash
tar -czf   "$HOME/backups/searxng-$(date +%F).tar.gz"   "$HOME/searxng"
```

## 26.6 Backup completo

```bash
tar -czf   "$HOME/backups/proyecto-ia-$(date +%F).tar.gz"   "$HOME/proyecto"   "$HOME/openwebui"   "$HOME/hermes"   "$HOME/opencode"   "$HOME/comfyui"   "$HOME/yolo"   "$HOME/searxng"
```

> Los modelos pueden ocupar cientos de GB. No siempre es conveniente incluirlos en cada backup. Para Ollama y ComfyUI se recomienda separar backup de configuración y backup de modelos.

---

# 27. Restauración

Ejemplo:

```bash
tar -xzf "$HOME/backups/openwebui-2026-10-06.tar.gz" -C /
```

Después:

```bash
docker compose -f docker-openwebui.yml up -d
```

Comprobar:

```bash
docker logs openwebui
```

---

# 28. Permisos

Comprobar propietario:

```bash
ls -ld   "$HOME/ollama"   "$HOME/openwebui"   "$HOME/hermes"   "$HOME/opencode"   "$HOME/comfyui"   "$HOME/yolo"
```

Si un contenedor no puede escribir:

```bash
sudo chown -R "$USER:$USER" "$HOME/openwebui"
```

Para Hermes, respetar el UID/GID que utilice la imagen y su configuración `PUID/PGID`; no ejecutar indiscriminadamente `chmod -R 777`.

---

# 29. Seguridad

## 29.1 No exponer innecesariamente servicios

Si el servidor solo se utiliza desde la propia máquina, publicar en localhost es más seguro:

```yaml
ports:
  - "127.0.0.1:3000:8080"
```

Si se necesita acceso LAN:

```yaml
ports:
  - "3000:8080"
```

## 29.2 Firewall

Docker puede interactuar de forma especial con las reglas de firewall. Docker advierte que los puertos publicados pueden saltarse determinadas reglas de UFW/firewalld y recomienda gestionar las reglas de filtrado Docker adecuadamente. citeturn0search1

No asumir que:

```bash
sudo ufw deny 3000
```

equivale automáticamente a impedir todo acceso a un puerto publicado por Docker.

## 29.3 Contraseñas

Nunca guardar:

```text
password123
admin
123456
```

Generar secretos:

```bash
openssl rand -hex 32
```

OpenCode recomienda `OPENCODE_SERVER_PASSWORD` para proteger el servidor web cuando se expone en red. citeturn3search1

---

# 30. Resolución de problemas

## 30.1 Docker no tiene acceso a la GPU

Comprobar host:

```bash
nvidia-smi
```

Comprobar toolkit:

```bash
nvidia-ctk --version
```

Comprobar Docker:

```bash
docker info | grep -i nvidia
```

Prueba directa:

```bash
docker run --rm --gpus all   nvidia/cuda:12.6.2-base-ubuntu24.04   nvidia-smi
```

Si falla, no continuar con ComfyUI, YOLO u Ollama GPU hasta solucionar este punto.

---

## 30.2 Open WebUI no encuentra Ollama

Comprobar:

```bash
docker exec openwebui curl -s   http://ollama:11434/api/tags
```

Si funciona, la red está bien.

Comprobar variable:

```bash
docker inspect openwebui   --format '{{range .Config.Env}}{{println .}}{{end}}'   | grep OLLAMA
```

Debe aparecer:

```text
OLLAMA_BASE_URL=http://ollama:11434
```

---

## 30.3 `localhost` no funciona entre contenedores

Incorrecto:

```text
http://localhost:11434
```

Correcto:

```text
http://ollama:11434
```

La razón es que cada contenedor tiene su propio namespace de red.

---

## 30.4 SearXNG no arranca

Logs:

```bash
docker logs searxng
```

Comprobar configuración:

```bash
cat configs/searxng/settings.yml
```

La documentación de SearXNG exige como mínimo una `server.secret_key` válida para una configuración mínima. citeturn1search7turn1search10

---

## 30.5 OpenCode no responde

```bash
docker logs opencode
```

Comprobar:

```bash
docker exec opencode opencode --version
```

Comprobar puerto:

```bash
ss -lntp | grep 8443
```

Desde host:

```bash
curl -I http://localhost:8443
```

---

## 30.6 ComfyUI se queda sin VRAM

Reducir tamaño del modelo, resolución o batch.

Comprobar:

```bash
watch -n 1 nvidia-smi
```

Comprobar logs:

```bash
docker logs -f comfyui
```

---

## 30.7 YOLO no utiliza GPU

Comprobar:

```bash
docker exec yolo python -c 'import torch; print(torch.cuda.is_available())'
```

Si devuelve:

```text
False
```

comprobar primero:

```bash
docker run --rm --gpus all   ultralytics/ultralytics:latest   python -c "import torch; print(torch.cuda.is_available())"
```

Ultralytics documenta el acceso de sus imágenes Docker a GPU NVIDIA mediante las opciones de GPU del runtime. citeturn1search0

---

# 31. Parada y arranque

Parar un servicio:

```bash
docker compose -f docker-ollama.yml down
```

Volver a arrancar:

```bash
docker compose -f docker-ollama.yml up -d
```

Parar todo:

```bash
for f in docker-*.yml; do
  docker compose --env-file .env -f "$f" down
done
```

> `down` no debe confundirse con `down -v`. En esta arquitectura los datos están en bind mounts del host, pero aun así se recomienda evitar comandos destructivos sin comprobar previamente qué recursos se van a eliminar.

---

# 32. Limpieza

Contenedores detenidos:

```bash
docker container prune
```

Imágenes sin usar:

```bash
docker image prune
```

Redes no utilizadas:

```bash
docker network prune
```

Todo lo no utilizado:

```bash
docker system prune
```

No utilizar:

```bash
docker system prune -a --volumes
```

sin verificar previamente, porque puede eliminar recursos que todavía se necesiten.

---

# 33. Verificación final

Ejecutar:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Verificar:

```text
ollama       -> 11434
openwebui    -> 3000
hermes-agent -> 8000
opencode     -> 8443
comfyui      -> 8188
yolo         -> 5000
searxng      -> 8080
```

Pruebas:

```bash
curl http://localhost:11434/api/tags
curl http://localhost:5000/health
curl http://localhost:8080
curl -I http://localhost:3000
curl -I http://localhost:8188
curl -I http://localhost:8443
```

GPU:

```bash
nvidia-smi
```

Docker GPU:

```bash
docker run --rm --gpus all   nvidia/cuda:12.6.2-base-ubuntu24.04   nvidia-smi
```

Red:

```bash
docker network inspect red-ia
```

---

# 34. Checklist de aceptación

## Sistema

- [ ] Ubuntu Server 24.04/26.04 instalado.
- [ ] NVIDIA driver funcionando.
- [ ] `nvidia-smi` funciona.
- [ ] Docker Engine instalado.
- [ ] Docker Compose v2 instalado.
- [ ] NVIDIA Container Toolkit instalado.
- [ ] Docker puede acceder a GPU.

## Red

- [ ] Existe `red-ia`.
- [ ] Todos los contenedores están conectados.
- [ ] Los nombres DNS internos resuelven.
- [ ] No existen puertos host duplicados.

## Servicios

- [ ] Ollama activo.
- [ ] Open WebUI activo.
- [ ] Hermes Agent activo.
- [ ] OpenCode activo.
- [ ] ComfyUI activo.
- [ ] YOLO activo.
- [ ] SearXNG activo.
- [ ] RAG configurado mediante Open WebUI/Ollama.

## Persistencia

- [ ] `$HOME/ollama`
- [ ] `$HOME/openwebui`
- [ ] `$HOME/hermes`
- [ ] `$HOME/opencode`
- [ ] `$HOME/comfyui`
- [ ] `$HOME/yolo`
- [ ] `$HOME/searxng`

## Seguridad

- [ ] `.env` tiene permisos `600`.
- [ ] Se han cambiado todos los secretos.
- [ ] OpenCode tiene contraseña.
- [ ] No se exponen puertos innecesarios a Internet.
- [ ] Se dispone de backup.

---

# 35. Referencia rápida

## Arranque

```bash
cd "$HOME/proyecto"

docker compose --env-file .env -f docker-ollama.yml up -d
docker compose --env-file .env -f docker-openwebui.yml up -d
docker compose --env-file .env -f docker-searxng.yml up -d
docker compose --env-file .env -f docker-comfyui.yml up -d
docker compose --env-file .env -f docker-yolo.yml up -d
docker compose --env-file .env -f docker-hermes-agent.yml up -d
docker compose --env-file .env -f docker-opencode.yml up -d
```

## Estado

```bash
docker ps
```

## Logs

```bash
docker logs -f NOMBRE
```

## GPU

```bash
nvidia-smi
```

## Reiniciar

```bash
docker restart NOMBRE
```

## Entrar

```bash
docker exec -it NOMBRE /bin/bash
```

Si la imagen no tiene Bash:

```bash
docker exec -it NOMBRE /bin/sh
```

---

# 36. URLs del proyecto

Desde la máquina host:

| Servicio | URL |
|---|---|
| Ollama | `http://IP:11434` |
| Open WebUI | `http://IP:3000` |
| Hermes | `http://IP:8000` |
| OpenCode | `http://IP:8443` |
| ComfyUI | `http://IP:8188` |
| YOLO | `http://IP:5000` |
| SearXNG | `http://IP:8080` |

Desde otros contenedores:

| Servicio | Endpoint interno |
|---|---|
| Ollama | `http://ollama:11434` |
| Open WebUI | `http://openwebui:8080` |
| Hermes | `http://hermes-agent:8642` |
| OpenCode | `http://opencode:4096` |
| ComfyUI | `http://comfyui:8188` |
| YOLO | `http://yolo:5000` |
| SearXNG | `http://searxng:8080` |

---

# 37. Fuentes técnicas consultadas

- Docker Engine — instalación oficial en Ubuntu.
- Docker Compose — plugin oficial.
- NVIDIA Container Toolkit — instalación y configuración de Docker.
- Open WebUI — instalación Docker y conexión con Ollama.
- Hermes Agent — despliegue Docker.
- OpenCode — servidor web y Docker.
- SearXNG — despliegue Docker.
- Ultralytics — Docker y GPU NVIDIA.
- ComfyUI Docker — persistencia y GPU.

Estas fuentes se han consultado para evitar fijar en el manual comandos obsoletos cuando la implementación actual difiere de la especificación original.
