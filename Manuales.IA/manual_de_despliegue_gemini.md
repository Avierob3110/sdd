# Manual de Instalación, Configuración y Operación: Pila de IA Local en Docker sobre Ubuntu

Este manual proporciona una guía paso a paso para desplegar y mantener una infraestructura local de Inteligencia Artificial utilizando Docker sobre Ubuntu Server con soporte para aceleración por GPU NVIDIA.

---

## 1. Prerrequisitos e Instalación Base

### 1.1. Actualización del Sistema
Actualiza el índice de paquetes e instala las dependencias base:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git build-essential ca-certificates gnupg lsb-release
```

### 1.2. Instalación de Drivers NVIDIA y CUDA
Agrega los repositorios oficiales e instala los controladores con soporte CUDA:

```bash
sudo apt install -y nvidia-driver-535 nvidia-dkms-535
```

Reinicia el sistema para aplicar los cambios:

```bash
sudo reboot
```

### 1.3. Instalación de Docker Engine y Docker Compose
Agrega el repositorio oficial de Docker e instala el motor junto al complemento de Compose:

```bash
# Agregar clave GPG oficial de Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmring -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Configurar repositorio
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Permitir ejecutar Docker sin sudo
sudo usermod -aG docker $USER
```

### 1.4. Instalación de NVIDIA Container Toolkit
Para permitir que los contenedores Docker accedan a la GPU física:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmring -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb [^ ]*#& [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg]#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

---

## 2. Estructura del Proyecto

Crea la estructura de carpetas necesaria en `$HOME/proyecto` junto a los volúmenes persistentes requeridos:

```bash
mkdir -p $HOME/proyecto/config
mkdir -p $HOME/ollama
mkdir -p $HOME/openwebui
mkdir -p $HOME/hermes
mkdir -p $HOME/opencode
mkdir -p $HOME/comfyui
mkdir -p $HOME/yolo
mkdir -p $HOME/searxng
mkdir -p $HOME/rag
```

Estructura de directorios resultante:

```text
$HOME/proyecto/
├── .env
├── docker-ollama.yml
├── docker-openwebui.yml
├── docker-hermes-agent.yml
├── docker-opencode.yml
├── docker-comfyui.yml
├── docker-yolo.yml
├── docker-searxng.yml
└── docker-rag.yml
```

---

## 3. Fichero de Entorno (`.env`)

Crea el archivo `$HOME/proyecto/.env` con la configuración centralizada de variables:

```env
# Configuración General
HOME_DIR=/home/usuario
DOCKER_NETWORK=red-ia

# Ollama
OLLAMA_PORT_HOST=11434
OLLAMA_PORT_CONTAINER=11434

# Open WebUI
OPENWEBUI_PORT_HOST=3000
OPENWEBUI_PORT_CONTAINER=8080

# Hermes Agent
HERMES_PORT_HOST=8000
HERMES_PORT_CONTAINER=8000

# OpenCode
OPENCODE_PORT_HOST=8443
OPENCODE_PORT_CONTAINER=8080

# ComfyUI
COMFYUI_PORT_HOST=8188
COMFYUI_PORT_CONTAINER=8188

# YOLO
YOLO_PORT_HOST=5000
YOLO_PORT_CONTAINER=5000

# SearXNG
SEARXNG_PORT_HOST=8080
SEARXNG_PORT_CONTAINER=8080
SEARXNG_SECRET_KEY=cambiar_por_una_clave_secreta_aleatoria
```

---

## 4. Ficheros de Configuración Docker Compose

Guarda cada archivo dentro del directorio `$HOME/proyecto/`.

### 4.1. `docker-ollama.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  ollama:
    container_name: ollama
    image: ollama/ollama:latest
    ports:
      - "${OLLAMA_PORT_HOST}:${OLLAMA_PORT_CONTAINER}"
    volumes:
      - ${HOME_DIR}/ollama:/root/.ollama
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped
```

### 4.2. `docker-openwebui.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  openwebui:
    container_name: openwebui
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "${OPENWEBUI_PORT_HOST}:${OPENWEBUI_PORT_CONTAINER}"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - ENABLE_SEARCH_AUGMENTATION=true
      - SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>
    volumes:
      - ${HOME_DIR}/openwebui:/app/backend/data
    networks:
      - red-ia
    restart: unless-stopped
```

### 4.3. `docker-hermes-agent.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  hermes-agent:
    container_name: hermes-agent
    image: python:3.11-slim
    command: tail -f /dev/null
    ports:
      - "${HERMES_PORT_HOST}:${HERMES_PORT_CONTAINER}"
    environment:
      - OLLAMA_HOST=http://ollama:11434
      - SEARXNG_HOST=http://searxng:8080
      - COMFYUI_HOST=http://comfyui:8188
    volumes:
      - ${HOME_DIR}/hermes:/app
    networks:
      - red-ia
    restart: unless-stopped
```

### 4.4. `docker-opencode.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  opencode:
    container_name: opencode
    image: lscr.io/linuxserver/code-server:latest
    ports:
      - "${OPENCODE_PORT_HOST}:${OPENCODE_PORT_CONTAINER}"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - ${HOME_DIR}/opencode:/config/workspace
    networks:
      - red-ia
    restart: unless-stopped
```

### 4.5. `docker-comfyui.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  comfyui:
    container_name: comfyui
    image: yananyi/comfyui:latest
    ports:
      - "${COMFYUI_PORT_HOST}:${COMFYUI_PORT_CONTAINER}"
    volumes:
      - ${HOME_DIR}/comfyui:/app/data
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped
```

### 4.6. `docker-yolo.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  yolo:
    container_name: yolo
    image: ultralytics/ultralytics:latest
    ports:
      - "${YOLO_PORT_HOST}:${YOLO_PORT_CONTAINER}"
    volumes:
      - ${HOME_DIR}/yolo:/ultralytics/data
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped
```

### 4.7. `docker-searxng.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  searxng:
    container_name: searxng
    image: searxng/searxng:latest
    ports:
      - "${SEARXNG_PORT_HOST}:${SEARXNG_PORT_CONTAINER}"
    environment:
      - SEARXNG_SECRET=${SEARXNG_SECRET_KEY}
    volumes:
      - ${HOME_DIR}/searxng:/etc/searxng
    networks:
      - red-ia
    restart: unless-stopped
```

### 4.8. `docker-rag.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  rag:
    container_name: rag
    image: alpine:latest
    command: tail -f /dev/null
    volumes:
      - ${HOME_DIR}/rag:/data
    networks:
      - red-ia
    restart: unless-stopped
```

---

## 5. Despliegue y Verificación

### 5.1. Crear la red Docker
Antes de levantar los contenedores, crea la red personalizada `red-ia`:

```bash
docker network create red-ia
```

### 5.2. Despliegue del Stack
Ejecuta los siguientes comandos desde dentro del directorio `$HOME/proyecto`:

```bash
cd $HOME/proyecto

# Desplegar los servicios individualmente o en lote
docker compose -f docker-ollama.yml up -d
docker compose -f docker-searxng.yml up -d
docker compose -f docker-openwebui.yml up -d
docker compose -f docker-hermes-agent.yml up -d
docker compose -f docker-opencode.yml up -d
docker compose -f docker-comfyui.yml up -d
docker compose -f docker-yolo.yml up -d
docker compose -f docker-rag.yml up -d
```

### 5.3. Verificación de Estado y Uso de GPU
Comprueba que los contenedores estén en ejecución:

```bash
docker ps
```

Verifica la disponibilidad de la GPU en los contenedores con aceleración habilitada (Ollama, ComfyUI, YOLO):

```bash
nvidia-smi
docker exec -it ollama nvidia-smi
```

### 5.4. URLs de Acceso a los Servicios
Accede desde tu navegador web utilizando la dirección IP de tu servidor:

| Servicio | URL Host |
| :--- | :--- |
| **Ollama API** | `http://<IP-SERVIDOR>:11434` |
| **Open WebUI** | `http://<IP-SERVIDOR>:3000` |
| **Hermes Agent** | `http://<IP-SERVIDOR>:8000` |
| **OpenCode** | `http://<IP-SERVIDOR>:8443` |
| **ComfyUI** | `http://<IP-SERVIDOR>:8188` |
| **YOLO** | `http://<IP-SERVIDOR>:5000` |
| **SearXNG** | `http://<IP-SERVIDOR>:8080` |

---

## 6. Mantenimiento y Actualización

### 6.1. Copias de Seguridad (Backups)
Para hacer una copia de seguridad de todos los volúmenes del sistema:

```bash
tar -czvf $HOME/backup_ia_$(date +%Y%m%d).tar.gz \
  $HOME/ollama \
  $HOME/openwebui \
  $HOME/hermes \
  $HOME/opencode \
  $HOME/comfyui \
  $HOME/yolo \
  $HOME/searxng \
  $HOME/rag
```

### 6.2. Actualización de Servicios
Para actualizar un servicio a su última versión:

```bash
cd $HOME/proyecto
docker compose -f docker-openwebui.yml pull
docker compose -f docker-openwebui.yml up -d
```

### 6.3. Resolución de Problemas Frecuentes (Permisos y Logs)
Si un contenedor falla al iniciar, revisa sus registros:

```bash
docker logs openwebui --tail 100 -f
```

Si hay problemas de escritura en los volúmenes montados en el Host, corrige la propiedad de los directorios:

```bash
sudo chown -R $USER:$USER $HOME/openwebui $HOME/opencode $HOME/comfyui $HOME/yolo $HOME/searxng $HOME/rag $HOME/hermes $HOME/ollama
```

---

## 7. Guía Interna de Integración de Servicios

Gracias a la red Docker `red-ia`, los contenedores se comunican internamente mediante el nombre asignado:

### 7.1. Conectar Ollama con Open WebUI
1. Entra en Open WebUI (`http://<IP-SERVIDOR>:3000`).
2. Ve a **Admin Settings > Connections > Ollama API**.
3. Establece la URL como: `http://ollama:11434`.

### 7.2. Conectar SearXNG con Open WebUI (Búsquedas en Web)
1. En Open WebUI, ve a **Admin Settings > Web Search**.
2. Selecciona **SearXNG** como motor de búsqueda.
3. Configura la URL del motor: `http://searxng:8080/search?q=<query>`.

### 7.3. Conectar Hermes Agent con Ollama, SearXNG y ComfyUI
Hermes Agent puede realizar llamadas a las API internas expuestas en la red Docker:
* **API Ollama:** `http://ollama:11434/api`
* **API SearXNG:** `http://searxng:8080/search`
* **API ComfyUI:** `http://comfyui:8188`

### 7.4. Integración de RAG con Open WebUI / Ollama
1. Los documentos depositados en `$HOME/rag` son accesibles para el procesamiento e indexación.
2. En Open WebUI, dirígete a la sección **Documents** e importa archivos. El sistema procesará las incrustaciones (*embeddings*) a través de `http://ollama:11434` utilizando el modelo cargado.