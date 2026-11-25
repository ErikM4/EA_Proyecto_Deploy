# EA_Proyecto_Deploy

Repositorio de orquestación mediante **Docker Compose** para el despliegue completo de la plataforma en producción sobre el servidor universitario (`192.168.10.157` / `10.4.120.117`) detrás del proxy inverso HTTPS.

## Arquitectura y Servicios

El ecosistema levanta 4 contenedores conectados a través de la red interna `ea-net`:

| Servicio | Contenedor | Imagen | Puerto Host → Contenedor | Dominio Público (HTTPS) |
| :--- | :--- | :--- | :--- | :--- |
| **Web Pública (React)** | `ea-proyecto-react` | `ghcr.io/guillermoaliaga/ea_proyecto_react:latest` | `8080` → `80` | `https://ea2.upc.edu` |
| **Backoffice (Angular)** | `ea-proyecto-angular` | `ghcr.io/guillermoaliaga/ea_proyecto_backoffice_frontend:latest` | `4000` → `80` | `https://ea2-back.upc.edu` |
| **API (Node/Express)** | `ea-proyecto-api` | `ghcr.io/erikm4/ea_proyecto_backend:latest` | `3000` → `1337` | `https://ea2-api.upc.edu` |
| **Base de Datos** | `ea-proyecto-mongodb` | `mongo:8` | Solo red interna (`27017`) | — |

---

## Requisitos Previos en el Servidor

* **Docker Engine** y **Docker Compose v2** instalados.
* Usuario añadido al grupo `docker` (`sudo usermod -aG docker $USER`) para ejecutar comandos sin `sudo`.
* Acceso a la red o VPN de la UPC para conectarse por SSH y acceder a los subdominios `*.upc.edu`.

---

## 1. Despliegue Inicial

1. **Conéctate al servidor por SSH y clona el repositorio:**

   ssh usuario@10.4.120.117
   git clone https://github.com/ErikM4/EA_Proyecto_Deploy.git
   cd EA_Proyecto_Deploy

2. **Crea el archivo de variables de entorno .env a partir de env.example:**

  cp env.example .env
  nano .env

3. **Asegúrate de configurar los orígenes CORS de producción y unas claves JWT seguras:**

MONGO_DB=ea_proyecto
CORS_ORIGIN=[https://ea2.upc.edu](https://ea2.upc.edu),[https://ea2-back.upc.edu](https://ea2-back.upc.edu)
JWT_SECRET=clave_secreta_larga_de_produccion_min_32_caracteres
JWT_REFRESH_SECRET=otra_clave_secreta_larga_de_produccion_min_32_caracteres
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

4. **Descarga las imágenes desde GitHub Container Registry y levanta los servicios:**

  docker compose pull
  docker compose up -d

5. **Inicialización de la Base de Datos (Seed)**

  docker run --rm --network ea-proyecto_ea-net \
  --env-file .env \
  -e MONGO_URL=mongodb://mongodb:27017/ea_proyecto \
  ghcr.io/erikm4/ea_proyecto_backend:latest \
  node build/seed.js

6. **3. Verificación en el Navegador**
Una vez levantados los contenedores y ejecutado el seed, verifica el funcionamiento en los siguientes enlaces públicos:

Web Pública (React): https://ea2.upc.edu

Backoffice (Angular): https://ea2-back.upc.edu

Documentación Swagger (API): https://ea2-api.upc.edu/api-docs