# 📰 Informe de Práctica: Despliegue en Producción con Docker y NGINX

## 1\. 🎯 Introducción y Objetivo

El objetivo de esta práctica fue migrar la arquitectura de la aplicación web completa (Frontend, Backend, y Base de Datos) de un entorno de desarrollo a un entorno de **Producción**, utilizando Docker. El foco principal fue optimizar la entrega del Frontend (React) mediante una estrategia de **Construcción de Múltiples Etapas (Multi-stage build)** para generar una imagen final ligera y eficiente servida por NGINX.

## 2\. 🏗️ Estrategia de Contenerización del Frontend

Para la aplicación Frontend (React), se adoptó una estrategia de **Construcción en Múltiples Etapas** para separar el entorno pesado de compilación (Node.js) del entorno ligero de ejecución (NGINX). Esto garantiza una imagen final optimizada.

### 2.1. Archivo `Dockerfile` (Multi-stage Build)

Se define un único `Dockerfile` que contiene dos etapas:

```dockerfile
# === ETAPA 1: BUILD (Construcción) ===
# Se utiliza una imagen Node.js para compilar la aplicación React.
FROM node:20-alpine AS builder

# Establecer directorio de trabajo
WORKDIR /app

# Copiar dependencias y ejecutarlas
COPY package.json package-lock.json ./
RUN npm install

# Copiar el código fuente
COPY . .

# Comando para generar los archivos estáticos de producción
RUN npm run build

# === ETAPA 2: PRODUCTION (Producción/Servicio) ===
# Se utiliza una imagen ligera de NGINX para servir los archivos estáticos.
FROM nginx:alpine

# Copiar el resultado de la construcción (archivos estáticos) desde la etapa 'builder'
# Los archivos estáticos se copian al directorio por defecto de NGINX.
COPY --from=builder /app/build /usr/share/nginx/html

# Copiar configuración personalizada de NGINX (si es necesario para rutas/proxy)
# COPY nginx/nginx.conf /etc/nginx/conf.d/default.conf

# Puerto de exposición (NGINX usa el puerto 80 por defecto)
EXPOSE 80

# El comando CMD por defecto de NGINX inicia el servidor.
```

## 3\. 🌐 Orquestación con Docker Compose (Producción)

Se actualizó el archivo `docker-compose.yml` para incluir la Base de Datos, el servicio Backend y el nuevo servicio Frontend basado en NGINX.

### 3.1. Archivo `docker-compose.yml`

```yaml
version: '3.8'

services:
  # 1. SERVICIO DE BASE DE DATOS (Ejemplo: PostgreSQL)
  db:
    image: postgres:16-alpine
    container_name: mi-db-prod
    restart: always
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mi_database
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - app-network

  # 2. SERVICIO DE BACKEND (API)
  backend:
    build: 
      context: ./backend
    container_name: mi-api-prod
    restart: always
    depends_on:
      - db # Dependencia para asegurar que la DB inicie primero
    ports:
      - "3001:3001"
    environment:
      # Configuración de conexión usando el nombre del servicio 'db'
      DATABASE_URL: postgres://user:password@db:5432/mi_database 
    networks:
      - app-network

  # 3. SERVICIO DE FRONTEND (NGINX - Producción)
  frontend:
    build:
      context: ./frontend # Ruta donde se encuentra el Dockerfile de Multi-stage
    container_name: mi-frontend-prod
    restart: always
    depends_on:
      - backend
    ports:
      - "80:80" # Mapeo del puerto 80 del NGINX al puerto 80 del host
    networks:
      - app-network
    # Nota: No se necesitan volúmenes de código en producción.
    # Las variables de entorno API_URL deben ser configuradas dentro del código JS
    # de React antes del build, o mediante la configuración de NGINX (si aplica).

# DEFINICIÓN DE RED Y VOLÚMENES
networks:
  app-network:
    driver: bridge

volumes:
  db-data:
```

### 3.2. Proceso de Despliegue

| **Acción** | **Comando Clave** | **Propósito** |
| :--- | :--- | :--- |
| **Despliegue Completo** | `docker-compose up --build -d` | Construye las imágenes necesarias (incluyendo el Frontend optimizado) y levanta todos los servicios en modo *detached*. |
| **Acceso a la App** | `http://localhost` o `http://localhost:80` | Acceso a la interfaz de usuario servida por el contenedor NGINX. |

## 4\. ✅ Conclusión

El uso de la **Construcción de Múltiples Etapas** fue esencial para pasar a producción. Se logró un tamaño de imagen del Frontend significativamente **reducido** (eliminando las dependencias de Node.js después de la compilación) y más seguro al servir la aplicación con el servidor web **NGINX**. Finalmente, Docker Compose facilitó la orquestación del *stack* completo (DB, API y Frontend en producción) bajo una única red interconectada.
