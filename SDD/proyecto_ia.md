# Proyecto becado SDD (): Despliegue de una pila de IA en Docker sobre Ubuntu

- **Versioón:** 1.0
- **Rol de creador:** Administrador se sistemas
- **Propósito:** Definir un manual técnico de requisitos y definiendo una arquitectura para la generación de un manual técnico de un manual técnico detallado con la instalación,configuración, tests y mantenimiento en formatomarkdown. (.md)

 # 1. Visión general del proyecto (Objetivo)
 El ejercicio del proyecto es desplegar una infraestructura de Inteligencia Artificial local utilizando contenedores Docker en un sistema operativo Ubuntu server. Cada servicio residirá en si propio contenedor docker. El sistema dispone de tarjeta gráfica NVIDIA (GPU).

 # 2. 



## 5. Instrucciones para generar un manual técnico
> **Instrucciones para la generación del documento de salida:**
> Actúa como un experto en administración de sistemas GNU/Linux y Devops, genera un **Manual de instalación,cinfiguración y operación** exhaustivo y detallado en formato Markdown basado en esta especificación.
> El manual generado debe incluir obligatoriamente las siguientes secciones:
> 1. **Prerequisitos e instalación base:** Comandos básicos de Linux oara instalar Docker, Docker compose,drivers de NVIDIA CUDA, utilizando repositorios apt.
> 2. **Estructura del proyecto:** Un `docker-<servicio>.yml` por cada uno de los servicios que vamos a montar donde <servicio> se sustituye por el nombre del contenedor.
> 3. **Fichero de configuración:** Fichero ´.env´ con todas las variables del entorno de todos los servicios.
> 4. **Fichero de entorno:** Fichero ´.env´ con todas las variables del entorno de todos los servicios.
> 5. **Despliegue y verificación:** Comandos relacionados con el arranque de los servicios ´(docker compose up -d)´, ocmprobación de logs, acceso a URLs de servicio, uso de la GPU `(nvidia-smi)`.
