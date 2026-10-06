---
title: "Manual de instalación, configuración y operación: pila de IA local en Docker sobre Ubuntu"
author: "Adaycv"
date: "2026-10-06"
category: "Despliegue IA"
tags: [markdown, ia, docker, nvidia, ollama, SDD]
---

# Manual de instalación, configuración y operación: pila de IA local en Docker sobre Ubuntu Server

- **Versión del manual:** 1.0 (generado a partir de la especificación SDD v1.0)
- **Destinatario:** administrador de sistemas con poca experiencia en Docker
- **Resultado final:** 8 contenedores (uno por servicio) en la red `red-ia`, usando la GPU NVIDIA donde el servicio lo permite.

---

## 0. Decisiones de diseño y correcciones a la especificación

La especificación original contiene algunas incoherencias que impedirían cumplir los criterios de aceptación (sin colisiones de puertos, ficheros funcionales). Se han resuelto así:

| # | Problema en la especificación | Decisión tomada |
|---|---|---|
| 1 | **RAG** usaba el puerto 11434, el mismo que Ollama (colisión). | RAG se implementa con **Qdrant** (base de datos vectorial) en el contenedor `rag`, puerto **6333**. Open WebUI lo usa para sus documentos, con embeddings generados por Ollama. |
| 2 | La tabla de DNS indicaba `http://openwebui:3000`, pero 3000 es el puerto del *host*. | Dentro de la red Docker se usa el puerto **interno**: `http://openwebui:8080`. |
| 3 | `hermesagent` en la tabla de DNS vs `hermes-agent` como contenedor. | Se usa `hermes-agent` (nombre del contenedor). |
| 4 | OpenCode: la tabla DNS usaba 8443 (externo). | Dentro de la red: `http://opencode:8080`. Desde el host: `http://IP:8443`. |
| 5 | Nombre de ficheros `docker-<servicio>.yml` (sección 5) vs `docker_<servicio>.yml` (sección 6). | Se usa **`docker-<servicio>.yml`** (guion medio). |
| 6 | SearXNG, Qdrant, OpenCode y Hermes Agent figuran con dependencia de GPU o se exige GPU en todos. | Solo usan GPU los servicios que **pueden** aprovecharla: Ollama, Open WebUI (imagen `cuda`), ComfyUI y YOLO. SearXNG, Qdrant, OpenCode y Hermes Agent no tienen carga de cálculo en GPU. |
| 7 | CUDA en el host. | En el host solo hace falta el **driver NVIDIA** y el **NVIDIA Container Toolkit**. Las librerías CUDA van dentro de las imágenes. No hay que instalar el CUDA Toolkit completo. |
| 8 | Hermes Agent: puerto interno 8000. | La imagen oficial expone su panel en **9119**. Se mapea `8000 (host) -> 9119 (contenedor)`. Ver sección 8, punto a validar. |

### Mapa final de puertos (sin colisiones)

| Servicio | Contenedor | Puerto interno | Puerto host | URL interna (red-ia) |
|---|---|---|---|---|
| Ollama | `ollama` | 11434 | 11434 | `http://ollama:11434` |
| Open WebUI | `openwebui` | 8080 | 3000 | `http://openwebui:8080` |
| Hermes Agent | `hermes-agent` | 9119 | 8000 | `http://hermes-agent:9119` |
| OpenCode | `opencode` | 8080 | 8443 | `http://opencode:8080` |
| ComfyUI | `comfyui` | 8188 | 8188 | `http://comfyui:8188` |
| YOLO | `yolo` | 5000 | 5000 | `http://yolo:5000` |
| SearXNG | `searxng` | 8080 | 8080 | `http://searxng:8080` |
| RAG (Qdrant) | `rag` | 6333 | 6333 | `http://rag:6333` |

Puertos de host usados: 3000, 5000, 6333, 8000, 8080, 8188, 8443, 11434. Todos distintos. Los puertos internos pueden repetirse (por ejemplo 8080) porque cada contenedor tiene su propia red virtual.

### Aviso importante sobre la memoria de la GPU (VRAM)

Con una RTX 3050 (6 u 8 GB) o una 4060 (8 GB), **Ollama, ComfyUI y YOLO compiten por la misma VRAM**. No pueden tener modelos grandes cargados a la vez. Este manual aplica estas medidas:

- Ollama descarga los modelos de la VRAM tras 2 minutos sin uso (`OLLAMA_KEEP_ALIVE=2m`) y carga solo uno a la vez (`OLLAMA_MAX_LOADED_MODELS=1`).
- ComfyUI arranca con `--lowvram`.
- YOLO usa el modelo pequeño `yolo11n.pt`.
- Recomendación práctica: no generes imágenes y chatees con un LLM grande **al mismo tiempo**.

---

## 1. Prerrequisitos e instalación base

> Todos los comandos se ejecutan en una terminal del servidor Ubuntu. `sudo` pide tu contraseña. Cuando veas `$` al inicio, no lo escribas.

### 1.1. Actualizar el sistema

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg git ufw htop tree
```

### 1.2. Instalar el driver de NVIDIA

Comprueba que el sistema ve la tarjeta:

```bash
lspci | grep -i nvidia
```

Instala el driver recomendado con el repositorio apt de Ubuntu:

```bash
sudo ubuntu-drivers list --gpgpu
sudo ubuntu-drivers install --gpgpu
sudo reboot
```

Tras reiniciar, verifica:

```bash
nvidia-smi
```

Debe aparecer una tabla con el nombre de tu GPU, la versión del driver y la línea `CUDA Version`. Si da error, no sigas: revisa la sección 6.4.

> Si usas Secure Boot, durante la instalación se te pedirá crear una contraseña (MOK) y confirmarla en el siguiente arranque, en una pantalla azul. Si no, el driver no cargará.

### 1.3. Instalar Docker Engine y Docker Compose desde el repositorio oficial

```bash
# 1) Clave GPG de Docker
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 2) Repositorio
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

# 3) Instalación
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

> **Si `apt update` da error 404 con el repositorio de Docker** (puede ocurrir justo tras salir una versión nueva de Ubuntu como la 26.04), usa temporalmente el paquete de Ubuntu: `sudo apt install -y docker.io docker-compose-v2 docker-buildx`. Elimina antes el fichero `/etc/apt/sources.list.d/docker.sources`.

Permite usar Docker sin `sudo` y comprueba la instalación:

```bash
sudo usermod -aG docker $USER
newgrp docker
docker --version
docker compose version
docker run --rm hello-world
```

### 1.4. Instalar el NVIDIA Container Toolkit (GPU dentro de Docker)

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Prueba definitiva de que Docker ve la GPU:

```bash
docker run --rm --gpus all nvidia/cuda:12.8.0-base-ubuntu24.04 nvidia-smi
```

Si ves la misma tabla que en el host, el entorno base está listo.

### 1.5. Firewall (opcional pero recomendado)

Abre solo lo necesario hacia tu red local (ajusta `192.168.1.0/24` a tu red):

```bash
sudo ufw allow OpenSSH
for p in 3000 5000 6333 8000 8080 8188 8443 11434; do
  sudo ufw allow from 192.168.1.0/24 to any port $p proto tcp
done
sudo ufw enable
```

> Importante: Docker modifica `iptables` por su cuenta y puede saltarse UFW para los puertos publicados. Para exponer solo a localhost, usa `127.0.0.1:PUERTO:PUERTO` en los ficheros yml. En una red doméstica de confianza el comportamiento por defecto es aceptable.

---

## 2. Estructura del proyecto

### 2.1. Crear los directorios

```bash
mkdir -p $HOME/proyecto/{comfyui,opencode,yolo}

# Datos persistentes (según la especificación, en $HOME)
mkdir -p $HOME/ollama
mkdir -p $HOME/openwebui
mkdir -p $HOME/hermes
mkdir -p $HOME/opencode/{config,workspace,share}
mkdir -p $HOME/comfyui/{input,output,user,custom_nodes}
mkdir -p $HOME/comfyui/models/{checkpoints,loras,vae,clip,unet,controlnet,upscale_models,embeddings,diffusion_models,text_encoders}
mkdir -p $HOME/yolo
mkdir -p $HOME/searxng/{config,cache}
mkdir -p $HOME/rag
mkdir -p $HOME/backups
```

### 2.2. Árbol resultante

```text
$HOME/
├── proyecto/                        <- ficheros de configuración (este manual)
│   ├── .env                         <- variables de entorno de todos los servicios
│   ├── docker-ollama.yml
│   ├── docker-openwebui.yml
│   ├── docker-hermes-agent.yml
│   ├── docker-opencode.yml
│   ├── docker-comfyui.yml
│   ├── docker-yolo.yml
│   ├── docker-searxng.yml
│   ├── docker-rag.yml
│   ├── start.sh                     <- arranca todo en orden
│   ├── stop.sh
│   ├── backup.sh
│   ├── update.sh
│   ├── comfyui/
│   │   └── Dockerfile
│   ├── opencode/
│   │   └── Dockerfile
│   └── yolo/
│       ├── Dockerfile
│       └── app.py
├── ollama/                          <- modelos LLM
├── openwebui/                       <- usuarios, chats, prompts, configuración
├── hermes/                          <- configuración de agentes
├── opencode/
│   ├── config/                      <- opencode.json
│   ├── workspace/                   <- código de los proyectos
│   └── share/                       <- sesiones y credenciales
├── comfyui/
│   ├── models/ (checkpoints, loras, vae, ...)
│   ├── input/  output/  user/  custom_nodes/
├── yolo/                            <- pesos, datasets, imágenes analizadas
├── searxng/
│   ├── config/settings.yml
│   └── cache/
├── rag/                             <- base vectorial (documentos indexados)
└── backups/
```

---

## 3. Ficheros de configuración (`docker-<servicio>.yml`)

### 3.1. Crear la red una sola vez

Todos los ficheros usan una red externa llamada `red-ia`, en modo bridge. Se crea antes de arrancar nada:

```bash
docker network create --driver bridge red-ia
docker network ls | grep red-ia
```

> Cada fichero declara `name: ia-<servicio>` en su primera línea. Así cada servicio es un "proyecto" independiente y puedes levantar, parar o actualizar uno sin afectar al resto, y Docker no avisa de "orphan containers".

Todos los comandos de las secciones siguientes se ejecutan desde `$HOME/proyecto`:

```bash
cd $HOME/proyecto
```

### 3.2. `docker-ollama.yml`

```yaml
name: ia-ollama

services:
  ollama:
    image: ollama/ollama:${OLLAMA_TAG}
    container_name: ollama
    hostname: ollama
    restart: unless-stopped
    ports:
      - "${OLLAMA_HOST_PORT}:11434"
    environment:
      - TZ=${TZ}
      - OLLAMA_HOST=0.0.0.0:11434
      - OLLAMA_KEEP_ALIVE=${OLLAMA_KEEP_ALIVE}
      - OLLAMA_MAX_LOADED_MODELS=${OLLAMA_MAX_LOADED_MODELS}
      - OLLAMA_NUM_PARALLEL=${OLLAMA_NUM_PARALLEL}
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    volumes:
      - ${HOME}/ollama:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    healthcheck:
      test: ["CMD", "ollama", "list"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 20s
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

### 3.3. `docker-openwebui.yml`

Incluye ya la integración con Ollama, SearXNG (búsqueda web), ComfyUI (imágenes) y Qdrant (RAG).

```yaml
name: ia-openwebui

services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:${OPENWEBUI_TAG}
    container_name: openwebui
    hostname: openwebui
    restart: unless-stopped
    ports:
      - "${OPENWEBUI_HOST_PORT}:8080"
    environment:
      - TZ=${TZ}
      - WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY}
      - WEBUI_NAME=${WEBUI_NAME}
      - ENABLE_SIGNUP=${OPENWEBUI_ENABLE_SIGNUP}
      # --- Ollama ---
      - OLLAMA_BASE_URL=http://ollama:11434
      # --- Búsqueda web con SearXNG ---
      - ENABLE_RAG_WEB_SEARCH=true
      - RAG_WEB_SEARCH_ENGINE=searxng
      - SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>
      - RAG_WEB_SEARCH_RESULT_COUNT=5
      # --- Generación de imágenes con ComfyUI ---
      - ENABLE_IMAGE_GENERATION=true
      - IMAGE_GENERATION_ENGINE=comfyui
      - COMFYUI_BASE_URL=http://comfyui:8188
      # --- RAG: Qdrant + embeddings de Ollama ---
      - VECTOR_DB=qdrant
      - QDRANT_URI=http://rag:6333
      - RAG_EMBEDDING_ENGINE=ollama
      - RAG_EMBEDDING_MODEL=${RAG_EMBEDDING_MODEL}
      - RAG_OLLAMA_BASE_URL=http://ollama:11434
      - NVIDIA_VISIBLE_DEVICES=all
    volumes:
      - ${HOME}/openwebui:/app/backend/data
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

### 3.4. `docker-hermes-agent.yml`

```yaml
name: ia-hermes-agent

services:
  hermes-agent:
    image: nousresearch/hermes-agent:${HERMES_TAG}
    container_name: hermes-agent
    hostname: hermes-agent
    restart: unless-stopped
    command: ["dashboard"]
    ports:
      - "${HERMES_HOST_PORT}:9119"
    environment:
      - TZ=${TZ}
      # Proveedor de modelos: Ollama como endpoint compatible con OpenAI
      - OPENAI_BASE_URL=http://ollama:11434/v1
      - OPENAI_API_KEY=ollama
    volumes:
      - ${HOME}/hermes:/opt/data
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

### 3.5. `docker-opencode.yml`

OpenCode no publica una imagen oficial para servidor web, por lo que se construye una.

`proyecto/opencode/Dockerfile`:

```dockerfile
FROM node:22-bookworm-slim

RUN apt-get update \
 && apt-get install -y --no-install-recommends git curl ca-certificates ripgrep openssh-client \
 && rm -rf /var/lib/apt/lists/*

RUN npm install -g opencode-ai

WORKDIR /workspace
EXPOSE 8080

CMD ["opencode", "web", "--hostname", "0.0.0.0", "--port", "8080"]
```

`docker-opencode.yml`:

```yaml
name: ia-opencode

services:
  opencode:
    build:
      context: ./opencode
    image: ia/opencode:local
    container_name: opencode
    hostname: opencode
    restart: unless-stopped
    ports:
      - "${OPENCODE_HOST_PORT}:8080"
    environment:
      - TZ=${TZ}
      - OPENCODE_SERVER_PASSWORD=${OPENCODE_SERVER_PASSWORD}
    volumes:
      - ${HOME}/opencode/config:/root/.config/opencode
      - ${HOME}/opencode/share:/root/.local/share/opencode
      - ${HOME}/opencode/workspace:/workspace
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

### 3.6. `docker-comfyui.yml`

`proyecto/comfyui/Dockerfile`:

```dockerfile
FROM pytorch/pytorch:2.7.1-cuda12.8-cudnn9-runtime

RUN apt-get update \
 && apt-get install -y --no-install-recommends git libgl1 libglib2.0-0 \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /opt/ComfyUI
RUN git clone --depth 1 https://github.com/comfyanonymous/ComfyUI.git . \
 && pip install --no-cache-dir -r requirements.txt

ENV COMFY_ARGS=""
EXPOSE 8188

CMD ["sh", "-c", "python main.py --listen 0.0.0.0 --port 8188 ${COMFY_ARGS}"]
```

`docker-comfyui.yml`:

```yaml
name: ia-comfyui

services:
  comfyui:
    build:
      context: ./comfyui
    image: ia/comfyui:local
    container_name: comfyui
    hostname: comfyui
    restart: unless-stopped
    ports:
      - "${COMFYUI_HOST_PORT}:8188"
    environment:
      - TZ=${TZ}
      - COMFY_ARGS=${COMFYUI_ARGS}
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    volumes:
      - ${HOME}/comfyui/models:/opt/ComfyUI/models
      - ${HOME}/comfyui/input:/opt/ComfyUI/input
      - ${HOME}/comfyui/output:/opt/ComfyUI/output
      - ${HOME}/comfyui/user:/opt/ComfyUI/user
      - ${HOME}/comfyui/custom_nodes:/opt/ComfyUI/custom_nodes
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

### 3.7. `docker-yolo.yml`

`proyecto/yolo/Dockerfile`:

```dockerfile
FROM ultralytics/ultralytics:latest

RUN pip install --no-cache-dir fastapi "uvicorn[standard]" python-multipart

COPY app.py /app/app.py

# /data es el volumen persistente: aquí se descargan los pesos del modelo
WORKDIR /data
ENV YOLO_CONFIG_DIR=/data/config

EXPOSE 5000
CMD ["uvicorn", "app:app", "--app-dir", "/app", "--host", "0.0.0.0", "--port", "5000"]
```

`proyecto/yolo/app.py`:

```python
import io
import os

import torch
from fastapi import FastAPI, File, UploadFile
from PIL import Image
from ultralytics import YOLO

app = FastAPI(title="YOLO API")
model = YOLO(os.getenv("YOLO_MODEL", "yolo11n.pt"))


@app.get("/health")
def health():
    return {"status": "ok", "gpu": torch.cuda.is_available()}


@app.post("/detect")
async def detect(file: UploadFile = File(...), conf: float = 0.25):
    image = Image.open(io.BytesIO(await file.read())).convert("RGB")
    result = model.predict(image, conf=conf, verbose=False)[0]
    detections = [
        {
            "class": result.names[int(box.cls)],
            "confidence": round(float(box.conf), 4),
            "box_xyxy": [round(v, 1) for v in box.xyxy[0].tolist()],
        }
        for box in result.boxes
    ]
    return {"count": len(detections), "detections": detections}
```

`docker-yolo.yml`:

```yaml
name: ia-yolo

services:
  yolo:
    build:
      context: ./yolo
    image: ia/yolo:local
    container_name: yolo
    hostname: yolo
    restart: unless-stopped
    ports:
      - "${YOLO_HOST_PORT}:5000"
    environment:
      - TZ=${TZ}
      - YOLO_MODEL=${YOLO_MODEL}
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    volumes:
      - ${HOME}/yolo:/data
    ipc: host
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

### 3.8. `docker-searxng.yml`

Antes de arrancarlo, crea `$HOME/searxng/config/settings.yml`. Es imprescindible habilitar el formato `json`, porque Open WebUI y OpenCode lo necesitan:

```bash
cat > $HOME/searxng/config/settings.yml <<'EOF'
use_default_settings: true

server:
  limiter: false
  image_proxy: true

search:
  safe_search: 0
  formats:
    - html
    - json
EOF
```

`docker-searxng.yml`:

```yaml
name: ia-searxng

services:
  searxng:
    image: searxng/searxng:${SEARXNG_TAG}
    container_name: searxng
    hostname: searxng
    restart: unless-stopped
    ports:
      - "${SEARXNG_HOST_PORT}:8080"
    environment:
      - TZ=${TZ}
      - SEARXNG_SECRET=${SEARXNG_SECRET}
      - SEARXNG_BASE_URL=http://localhost:${SEARXNG_HOST_PORT}/
    volumes:
      - ${HOME}/searxng/config:/etc/searxng
      - ${HOME}/searxng/cache:/var/cache/searxng
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

### 3.9. `docker-rag.yml` (Qdrant como almacén vectorial)

```yaml
name: ia-rag

services:
  rag:
    image: qdrant/qdrant:${QDRANT_TAG}
    container_name: rag
    hostname: rag
    restart: unless-stopped
    ports:
      - "${RAG_HOST_PORT}:6333"
    environment:
      - TZ=${TZ}
    volumes:
      - ${HOME}/rag:/qdrant/storage
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

---

## 4. Fichero de entorno (`.env`)

Crea `$HOME/proyecto/.env`. Este fichero contiene contraseñas: protégelo con `chmod 600`.

```bash
cd $HOME/proyecto
cat > .env <<EOF
# ---------- General ----------
TZ=Europe/Madrid

# ---------- Versiones de imagen ----------
OLLAMA_TAG=latest
OPENWEBUI_TAG=cuda
HERMES_TAG=latest
SEARXNG_TAG=latest
QDRANT_TAG=latest

# ---------- Puertos del host ----------
OLLAMA_HOST_PORT=11434
OPENWEBUI_HOST_PORT=3000
HERMES_HOST_PORT=8000
OPENCODE_HOST_PORT=8443
COMFYUI_HOST_PORT=8188
YOLO_HOST_PORT=5000
SEARXNG_HOST_PORT=8080
RAG_HOST_PORT=6333

# ---------- Ollama ----------
OLLAMA_KEEP_ALIVE=2m
OLLAMA_MAX_LOADED_MODELS=1
OLLAMA_NUM_PARALLEL=1

# ---------- Open WebUI ----------
WEBUI_NAME=IA Local
WEBUI_SECRET_KEY=$(openssl rand -hex 32)
OPENWEBUI_ENABLE_SIGNUP=true
RAG_EMBEDDING_MODEL=nomic-embed-text

# ---------- SearXNG ----------
SEARXNG_SECRET=$(openssl rand -hex 32)

# ---------- OpenCode ----------
OPENCODE_SERVER_PASSWORD=$(openssl rand -hex 12)

# ---------- ComfyUI ----------
COMFYUI_ARGS=--lowvram

# ---------- YOLO ----------
YOLO_MODEL=yolo11n.pt
EOF

chmod 600 .env
```

> El bloque anterior genera automáticamente las claves aleatorias con `openssl`. Para ver la contraseña de OpenCode: `grep OPENCODE_SERVER_PASSWORD .env`.
>
> `OPENWEBUI_ENABLE_SIGNUP=true` permite crear la cuenta de administrador (el primer usuario registrado lo será). Después de crearla, cámbialo a `false` y recrea el contenedor.

---

## 5. Despliegue y verificación

### 5.1. Scripts de arranque y parada

`$HOME/proyecto/start.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"

docker network inspect red-ia >/dev/null 2>&1 || docker network create --driver bridge red-ia

# Orden: primero los servicios de los que dependen otros
for svc in ollama rag searxng comfyui yolo openwebui hermes-agent opencode; do
  echo ">>> Arrancando $svc"
  docker compose --env-file .env -f "docker-${svc}.yml" up -d --build
done

docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

`$HOME/proyecto/stop.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"
for svc in opencode hermes-agent openwebui yolo comfyui searxng rag ollama; do
  docker compose --env-file .env -f "docker-${svc}.yml" down
done
```

```bash
chmod +x start.sh stop.sh backup.sh update.sh 2>/dev/null || chmod +x start.sh stop.sh
```

### 5.2. Preparar permisos antes del primer arranque

SearXNG se ejecuta con un usuario sin privilegios y necesita escribir en sus carpetas:

```bash
sudo chown -R 977:977 $HOME/searxng
```

### 5.3. Validar la sintaxis de los ficheros

```bash
cd $HOME/proyecto
for f in docker-*.yml; do
  docker compose --env-file .env -f "$f" config -q && echo "OK  $f" || echo "ERROR $f"
done
```

Todos deben mostrar `OK`. La primera vez no hay nada que corregir si copiaste los ficheros tal cual.

### 5.4. Arrancar

La primera ejecución descarga imágenes (varios GB) y construye ComfyUI, YOLO y OpenCode. Puede tardar 10 a 30 minutos.

```bash
cd $HOME/proyecto
./start.sh
```

Para arrancar un servicio individual:

```bash
docker compose --env-file .env -f docker-ollama.yml up -d
```

### 5.5. Descargar los modelos de Ollama

```bash
# Modelo de chat (cabe en 6-8 GB de VRAM)
docker exec -it ollama ollama pull qwen3:8b
# Alternativa más ligera para 6 GB
docker exec -it ollama ollama pull llama3.2:3b
# Modelo de embeddings para RAG (obligatorio para la integración RAG)
docker exec -it ollama ollama pull nomic-embed-text

docker exec -it ollama ollama list
```

### 5.6. Descargar un modelo para ComfyUI

ComfyUI necesita al menos un *checkpoint* en `$HOME/comfyui/models/checkpoints/` (ficheros `.safetensors`). Descárgalo desde Hugging Face o Civitai con `wget` o `curl -L -o`. Para 6-8 GB de VRAM, elige modelos basados en SD 1.5 o SDXL. Después, en ComfyUI, pulsa `Refresh` (tecla `r`).

### 5.7. Comprobar el estado y los logs

```bash
docker ps
docker logs --tail 50 ollama
docker logs -f openwebui            # Ctrl+C para salir
docker logs --tail 50 comfyui
docker compose -f docker-searxng.yml logs --tail 50
```

Todos los contenedores deben figurar como `Up`. Si alguno aparece como `Restarting`, mira sus logs (sección 6.4).

### 5.8. Acceso a las URLs (sustituye `IP_SERVIDOR`)

| Servicio | URL desde tu navegador | Qué deberías ver |
|---|---|---|
| Open WebUI | `http://IP_SERVIDOR:3000` | Pantalla de creación de la cuenta de administrador |
| Hermes Agent | `http://IP_SERVIDOR:8000` | Panel de Hermes |
| OpenCode | `http://IP_SERVIDOR:8443` | Interfaz web; usuario `opencode`, contraseña = `OPENCODE_SERVER_PASSWORD` |
| ComfyUI | `http://IP_SERVIDOR:8188` | Editor de nodos |
| YOLO | `http://IP_SERVIDOR:5000/docs` | Documentación interactiva de la API |
| SearXNG | `http://IP_SERVIDOR:8080` | Buscador |
| Ollama | `http://IP_SERVIDOR:11434` | Texto `Ollama is running` |
| RAG (Qdrant) | `http://IP_SERVIDOR:6333/dashboard` | Panel de Qdrant |

Averigua la IP del servidor con `hostname -I`.

### 5.9. Verificar el uso de la GPU

```bash
nvidia-smi                   # una vez
watch -n 1 nvidia-smi        # actualización continua (Ctrl+C para salir)
```

Prueba de carga real con Ollama (mantén `nvidia-smi` abierto en otra terminal):

```bash
docker exec -it ollama ollama run qwen3:8b "Explica qué es Docker en dos frases"
docker exec -it ollama ollama ps
```

`ollama ps` debe mostrar `100% GPU` en la columna `PROCESSOR`. Si indica `CPU`, la GPU no está accesible: ver sección 6.4.

Otras comprobaciones:

```bash
# GPU visible en cada contenedor que la usa
docker exec comfyui nvidia-smi -L
docker exec yolo nvidia-smi -L
docker exec openwebui nvidia-smi -L

# YOLO confirma que usa GPU
curl -s http://localhost:5000/health
# Esperado: {"status":"ok","gpu":true}
```

### 5.10. Tests funcionales rápidos

```bash
# Ollama: API
curl -s http://localhost:11434/api/tags

# Ollama: generación
curl -s http://localhost:11434/api/generate \
  -d '{"model":"llama3.2:3b","prompt":"Hola","stream":false}'

# SearXNG: debe devolver JSON (si devuelve 403, falta "json" en settings.yml)
curl -s "http://localhost:8080/search?q=ubuntu&format=json" | head -c 300

# YOLO: detección sobre una imagen
curl -s -X POST -F "file=@/ruta/foto.jpg" http://localhost:5000/detect

# Qdrant
curl -s http://localhost:6333/collections
```

Comprobar la red y el DNS interno (un contenedor temporal dentro de `red-ia`):

```bash
for url in http://ollama:11434 http://openwebui:8080 http://searxng:8080 \
           http://comfyui:8188 http://yolo:5000/health http://rag:6333 \
           http://opencode:8080 http://hermes-agent:9119; do
  printf "%-32s" "$url"
  docker run --rm --network red-ia curlimages/curl:latest \
    -s -o /dev/null -w "%{http_code}\n" --max-time 5 "$url" || echo "FALLO"
done
```

Un código `200`, `301`, `302` o `401` indica que el servicio responde. `000` o `FALLO` indica que no está accesible.

---

## 6. Mantenimiento y actualización

### 6.1. Copias de seguridad

`$HOME/proyecto/backup.sh`. Para los servicios que escriben en disco, copia y los vuelve a arrancar:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"

DEST="$HOME/backups"
FECHA=$(date +%Y%m%d_%H%M)
mkdir -p "$DEST"

echo ">>> Parando servicios con estado para una copia consistente"
for svc in openwebui rag hermes-agent opencode; do
  docker compose --env-file .env -f "docker-${svc}.yml" stop
done

# Datos pequeños e importantes (siempre)
tar -czf "$DEST/config_${FECHA}.tar.gz" -C "$HOME" \
  openwebui rag hermes opencode searxng/config yolo proyecto/.env proyecto/*.yml \
  proyecto/comfyui proyecto/opencode proyecto/yolo

# Datos grandes y recuperables (modelos): solo si lo necesitas
# tar -czf "$DEST/modelos_${FECHA}.tar.gz" -C "$HOME" ollama comfyui/models

echo ">>> Reiniciando servicios"
for svc in rag openwebui hermes-agent opencode; do
  docker compose --env-file .env -f "docker-${svc}.yml" start
done

# Conservar solo las 7 últimas copias de configuración
ls -1t "$DEST"/config_*.tar.gz | tail -n +8 | xargs -r rm --
echo ">>> Copia creada en $DEST/config_${FECHA}.tar.gz"
```

Ejecutar y automatizar (cada domingo a las 03:00):

```bash
chmod +x $HOME/proyecto/backup.sh
$HOME/proyecto/backup.sh
crontab -e
# Añadir la línea:
# 0 3 * * 0 $HOME/proyecto/backup.sh >> $HOME/backups/backup.log 2>&1
```

Restaurar:

```bash
cd $HOME/proyecto && ./stop.sh
tar -xzf $HOME/backups/config_FECHA.tar.gz -C $HOME
./start.sh
```

> Los modelos de Ollama y ComfyUI ocupan muchos GB y se pueden volver a descargar. Haz copia de ellos solo si tu conexión es lenta.

### 6.2. Actualización de servicios

`$HOME/proyecto/update.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"

./backup.sh

# Imágenes remotas
for svc in ollama openwebui hermes-agent searxng rag; do
  docker compose --env-file .env -f "docker-${svc}.yml" pull
  docker compose --env-file .env -f "docker-${svc}.yml" up -d
done

# Imágenes construidas localmente: reconstruir sin caché para traer la última versión
for svc in comfyui yolo opencode; do
  docker compose --env-file .env -f "docker-${svc}.yml" build --pull --no-cache
  docker compose --env-file .env -f "docker-${svc}.yml" up -d
done

docker image prune -f
docker ps --format 'table {{.Names}}\t{{.Status}}'
```

Actualizar un único servicio:

```bash
docker compose --env-file .env -f docker-ollama.yml pull
docker compose --env-file .env -f docker-ollama.yml up -d
```

Actualizar modelos de Ollama:

```bash
docker exec ollama ollama pull qwen3:8b
```

Actualizar el driver NVIDIA (tras `apt upgrade`, reinicia si el kernel o el driver cambian):

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

> Si una actualización rompe algo, fija la versión anterior en `.env` (por ejemplo `OLLAMA_TAG=0.12.0`) y ejecuta `up -d` de nuevo. Consulta los *tags* disponibles en Docker Hub o GHCR.

### 6.3. Limpieza y monitorización

```bash
docker system df                 # espacio usado por Docker
docker image prune -f            # imágenes huérfanas
docker builder prune -f          # caché de construcción
docker stats --no-stream         # CPU y RAM por contenedor
df -h $HOME                      # disco
```

### 6.4. Resolución de errores comunes

**a) Errores de permisos (`Permission denied`)**

| Síntoma | Causa | Solución |
|---|---|---|
| `permission denied` al ejecutar `docker` | Tu usuario no está en el grupo docker | `sudo usermod -aG docker $USER` y cierra sesión / `newgrp docker` |
| SearXNG no arranca o no guarda | La carpeta no pertenece a su usuario (977) | `sudo chown -R 977:977 $HOME/searxng` |
| Ficheros en `$HOME/hermes`, `$HOME/openwebui` o `$HOME/comfyui` que no puedes borrar/editar | Los contenedores escriben como `root` | `sudo chown -R $USER:$USER $HOME/hermes` (equivalente para cada carpeta) |
| Hermes Agent da errores de escritura en `/opt/data` | El usuario interno no puede escribir en la carpeta | `sudo chown -R $USER:$USER $HOME/hermes` y, si persiste, `chmod -R u+rwX,g+rwX $HOME/hermes` |
| Los datos desaparecen tras reiniciar | Volumen mal mapeado | Revisa con `docker inspect NOMBRE \| grep -A5 Mounts` |

**b) La GPU no se detecta**

```bash
nvidia-smi                                   # ¿funciona en el host?
docker run --rm --gpus all nvidia/cuda:12.8.0-base-ubuntu24.04 nvidia-smi
cat /etc/docker/daemon.json                  # debe contener el runtime "nvidia"
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

- Si falla `nvidia-smi` en el host: el driver no está cargado. Reinstala (`sudo ubuntu-drivers install --gpgpu`), reinicia y revisa Secure Boot.
- Si funciona en el host pero no en Docker: falta o está mal configurado el Container Toolkit (sección 1.4).
- Error `Failed to initialize NVML: Unknown Error` tras un tiempo funcionando: ocurre a veces con cgroups. Reinicia el contenedor (`docker restart ollama`).

**c) Puerto ocupado (`port is already allocated` / `address already in use`)**

```bash
sudo ss -tulpn | grep :8080
```

Cambia el puerto correspondiente en `.env` y recrea el servicio con `up -d`.

**d) Falta de VRAM (`CUDA out of memory`)**

- Para el servicio que no estés usando: `docker compose -f docker-comfyui.yml stop`.
- Usa un modelo de Ollama más pequeño (`llama3.2:3b`).
- Ejecuta `docker exec ollama ollama ps` para ver qué modelo ocupa memoria y `ollama stop NOMBRE` para liberarla.
- En ComfyUI, mantén `--lowvram` o prueba `--novram` en `COMFYUI_ARGS`.

**e) Open WebUI no ve los modelos de Ollama**

```bash
docker exec openwebui sh -c 'curl -s http://ollama:11434/api/tags | head -c 200'
```

Si falla, comprueba que ambos contenedores están en `red-ia` con `docker network inspect red-ia`. Con `docker-compose` por separado, un contenedor sin la sección `networks` queda en otra red.

**f) SearXNG responde 403 a `format=json`**

Falta `json` en `search.formats` de `$HOME/searxng/config/settings.yml`. Corrígelo y ejecuta `docker restart searxng`.

**g) Un contenedor se reinicia en bucle**

```bash
docker logs --tail 100 NOMBRE
docker inspect NOMBRE --format '{{.State.ExitCode}} {{.State.Error}}'
```

**h) Se agota el disco**

Los modelos son los grandes consumidores. Revisa con `du -sh $HOME/ollama $HOME/comfyui/models` y elimina modelos de Ollama que no uses con `docker exec ollama ollama rm NOMBRE`.

---

## 7. Guía interna de integración entre servicios

Regla de oro: **dentro de la red `red-ia` se usa siempre el nombre del contenedor y el puerto interno** (por ejemplo `http://ollama:11434`), nunca `localhost` ni la IP del host. En un contenedor, `localhost` es el propio contenedor.

### 7.1. Ollama con Open WebUI (ya configurado)

La variable `OLLAMA_BASE_URL=http://ollama:11434` en `docker-openwebui.yml` hace la conexión. Verificación:

1. Entra en `http://IP_SERVIDOR:3000` y crea la cuenta de administrador.
2. Los modelos descargados aparecen en el selector de arriba a la izquierda.
3. Si no: `Panel de administración > Ajustes > Conexiones > Ollama` y comprueba la URL `http://ollama:11434`.

### 7.2. SearXNG con Open WebUI (búsqueda web)

Ya está definido por entorno (`RAG_WEB_SEARCH_ENGINE=searxng` y `SEARXNG_QUERY_URL`). Para usarlo:

1. `Panel de administración > Ajustes > Búsqueda web`: verifica que está activada y el motor es `searxng`.
2. En un chat, pulsa el icono `+` y activa **Búsqueda web**.
3. Prueba: "¿Cuál es la última versión de Ubuntu Server?".

### 7.3. ComfyUI con Open WebUI (generación de imágenes)

La URL se define por entorno (`COMFYUI_BASE_URL=http://comfyui:8188`), pero Open WebUI necesita además un **flujo de trabajo (workflow)** de ComfyUI:

1. En ComfyUI abre un flujo de texto a imagen y usa `Workflow > Export (API)` para obtener el JSON.
2. En Open WebUI: `Panel de administración > Ajustes > Imágenes`.
3. Motor `ComfyUI`, URL `http://comfyui:8188`, pega el JSON del flujo.
4. Asigna cada nodo (prompt, ancho, alto, semilla, modelo...) con los ID de nodo que muestra la pantalla.
5. Guarda y prueba con el botón de imagen en un chat.

### 7.4. RAG (Qdrant) con Open WebUI y Ollama

Ya configurado: Open WebUI guarda los vectores en `http://rag:6333` y calcula los embeddings con `nomic-embed-text` en Ollama.

1. Asegúrate de haber ejecutado `ollama pull nomic-embed-text` (sección 5.5).
2. En Open WebUI: `Espacio de trabajo > Conocimiento > Nuevo`, y sube documentos (PDF, TXT, MD...). También puedes copiar ficheros a `$HOME/rag` para uso propio, pero Open WebUI gestiona la indexación desde su interfaz.
3. En un chat, escribe `#` y selecciona la colección para consultarla.
4. Comprobación: `curl -s http://localhost:6333/collections` debe listar colecciones tras indexar algo.

### 7.5. Ollama con OpenCode

OpenCode se configura mediante un fichero `opencode.json`. Crea `$HOME/opencode/config/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": {
        "baseURL": "http://ollama:11434/v1"
      },
      "models": {
        "qwen3:8b": { "name": "Qwen3 8B" },
        "llama3.2:3b": { "name": "Llama 3.2 3B" }
      }
    }
  },
  "model": "ollama/qwen3:8b"
}
```

Reinicia (`docker restart opencode`) y elige el modelo en la interfaz.

> Para programación, los modelos pequeños dan resultados limitados. Si tu GPU lo permite, aumenta el contexto de Ollama (`OLLAMA_CONTEXT_LENGTH=16384` en `docker-ollama.yml`), teniendo en cuenta que consume más VRAM.

### 7.6. SearXNG con OpenCode

OpenCode no trae SearXNG de serie. Se integra mediante un servidor MCP. Añade a `opencode.json` (fusionando con el contenido anterior):

```json
{
  "mcp": {
    "searxng": {
      "type": "local",
      "command": ["npx", "-y", "mcp-searxng"],
      "environment": {
        "SEARXNG_URL": "http://searxng:8080"
      },
      "enabled": true
    }
  }
}
```

La imagen de OpenCode ya incluye Node.js, necesario para `npx`. Reinicia el contenedor y comprueba que la herramienta `searxng` aparece activa en la lista de servidores MCP.

### 7.7. Ollama y SearXNG con Hermes Agent

Ollama ya está declarado por entorno (`OPENAI_BASE_URL=http://ollama:11434/v1`). Para fijar el modelo, ejecuta el asistente interactivo una vez:

```bash
cd $HOME/proyecto
docker compose --env-file .env -f docker-hermes-agent.yml run --rm hermes-agent setup
```

- Proveedor: endpoint personalizado / compatible con OpenAI.
- URL base: `http://ollama:11434/v1`
- Clave API: `ollama` (cualquier texto)
- Modelo: `qwen3:8b`

Después, reinicia: `docker restart hermes-agent`. La configuración se guarda en `$HOME/hermes`. Para añadir búsqueda web con SearXNG, usa su compatibilidad con MCP, apuntando a `http://searxng:8080` como en el punto 7.6, siguiendo la documentación de Hermes Agent (`https://hermes-agent.nousresearch.com/docs`).

> Hermes Agent funciona mucho mejor con modelos grandes. Con 8 GB de VRAM, espera resultados limitados en tareas complejas.

### 7.8. Hermes Agent con ComfyUI y YOLO

Estos servicios exponen APIs HTTP que un agente puede llamar como herramientas:

- ComfyUI: `http://comfyui:8188` (API `/prompt`, con un flujo exportado en formato API).
- YOLO: `POST http://yolo:5000/detect` con una imagen en el campo `file`.

Prueba de conectividad desde dentro de la red:

```bash
docker exec hermes-agent sh -c 'curl -s http://yolo:5000/health || wget -qO- http://yolo:5000/health'
```

### 7.9. Resumen de dependencias

```text
                 +--> ollama <---+--- hermes-agent
openwebui -------+--> searxng    +--- opencode (MCP --> searxng)
    |            +--> comfyui
    |            +--> rag (Qdrant) --- embeddings --> ollama
    
yolo  : API independiente (la usan agentes o scripts)
```

---

## 8. Puntos a validar antes de poner en producción

Estos elementos dependen de proyectos que evolucionan con rapidez; compruébalos en la primera instalación:

1. **Hermes Agent:** la imagen oficial es `nousresearch/hermes-agent` y guarda su estado en `/opt/data`. El puerto interno `9119` del comando `dashboard` procede de la documentación de la comunidad y puede variar. Compruébalo con `docker logs hermes-agent`. Si el panel escucha en otro puerto o solo en `127.0.0.1`, ajusta el mapeo de puertos y el `command`. Consulta las opciones con `docker run --rm nousresearch/hermes-agent dashboard --help`.
2. **OpenCode:** se instala mediante npm (`opencode-ai`) y el modo servidor web es `opencode web`. Si cambian sus parámetros, revisa `docker run --rm ia/opencode:local opencode web --help`.
3. **Ubuntu 26.04:** si algún repositorio externo (Docker, NVIDIA) aún no publica paquetes para la versión nueva, usa Ubuntu 24.04 LTS, soportada por todos ellos.
4. **Seguridad:** la pila no tiene HTTPS ni autenticación en ComfyUI, YOLO, SearXNG, Ollama ni Qdrant. Úsala solo en red local de confianza o añade un *reverse proxy* (Caddy o Nginx Proxy Manager) con contraseña antes de exponerla a Internet.

---

## 9. Lista de comprobación de criterios de aceptación

| Criterio | Cómo se cumple | Cómo comprobarlo |
|---|---|---|
| 1. Ficheros yml funcionales | Un fichero por servicio, validados con `docker compose config` | Sección 5.3 |
| 2. GPU / CUDA | Ollama, Open WebUI (`:cuda`), ComfyUI y YOLO reservan la GPU. Los demás no tienen cálculo en GPU | Sección 5.9 |
| 3. Sin colisiones de puertos | 8 puertos de host distintos (RAG pasa a 6333) | `docker ps` y sección 0 |
| 4. Paso a paso para novatos | Comandos completos, verificación tras cada bloque y tabla de errores | Secciones 1 a 6 |

