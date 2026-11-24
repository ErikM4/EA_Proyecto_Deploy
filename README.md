# EA_Proyecto_Deploy

Orquestación del proyecto de Ingeniería de Aplicaciones (EA): un solo `docker compose` levanta la base de datos MongoDB, la API REST con JWT y Socket.IO, el backoffice en Angular y la web pública en React.

Cada aplicación vive en su propio repositorio y publica automáticamente su imagen Docker en **GitHub Container Registry (GHCR)** en cada push a `main`. Este repositorio no contiene código de aplicación: únicamente compone y configura esas imágenes.

## Servicios

| Servicio | Imagen | Puerto en tu máquina | Descripción |
|---|---|---|---|
| `mongodb` | `mongo:8` | interno (`27017`) | Base de datos MongoDB persistente |
| `api` | `ghcr.io/erikm4/ea_proyecto_backend:latest` | `1337` | API REST + WebSockets (Socket.IO) |
| `angular` | `ghcr.io/guillermoaliaga/ea_proyecto_backoffice_frontend:latest` | `4200` | Backoffice de administración en Angular |
| `react` | `ghcr.io/guillermoaliaga/ea_proyecto_react:latest` | `4300` | Aplicación web pública en React |

Los cuatro servicios comparten la red interna `ea-net`, de modo que la API se comunica con MongoDB usando el nombre de servicio `mongodb`.

---

## Cómo arrancarlo

1. Clona este repositorio y crea el fichero de entorno a partir de la plantilla:

```bash
cp env.example .env
```

2. Edita `.env` y configura tus claves para `JWT_SECRET` y `JWT_REFRESH_SECRET` (mínimo 32 caracteres recomendados).
3. Descarga las últimas imágenes y levanta los contenedores:

```bash
docker compose pull
docker compose up -d
```

Una vez levantados los servicios:

* **Web Pública (React):** <http://localhost:4300>
* **Backoffice (Angular):** <http://localhost:4200>
* **API REST:** <http://localhost:1337> (Documentación Swagger en <http://localhost:1337/api-docs>)

> **Nota de seguridad:** El `docker-compose.yml` no arrancará si faltan `JWT_SECRET` o `JWT_REFRESH_SECRET` en el `.env`, evitando levantar la API con claves por defecto.

> **Visibilidad de imágenes en GHCR:** Los paquetes en `ghcr.io` deben estar configurados como **Public** (en GitHub > *Packages* > *Package settings* > *Change visibility*). Si alguno es privado, será necesario ejecutar antes `docker login ghcr.io` con un token con permiso `read:packages`.

---

## Inicialización de datos (Seed)

Al arrancar el proyecto por primera vez, la base de datos MongoDB estará vacía. Los datos de prueba y el script de inicialización (`npm run seed`) se encuentran en el repositorio del backend (`EA_Proyecto_Backend`).

Como el contenedor `mongodb` no expone el puerto `27017` al host por seguridad, el script `seed` se ejecuta mediante un contenedor efímero conectado directamente a la red interna `ea-proyecto_ea-net`, apuntando a la misma base de datos (`MONGO_DB`) definida en tu `.env` (por defecto `ea_proyecto`).

### En Windows (PowerShell)

Situado en la carpeta donde tengas clonados los repositorios (o ajustando la ruta al backend):

```powershell
docker run --rm --network ea-proyecto_ea-net `
  -v "${PWD}\..\EA_Proyecto_Backend:/app" `
  -v /app/node_modules `
  -w /app `
  -e JWT_SECRET=secreto-temporal-de-32-caracteres-para-seed `
  -e JWT_REFRESH_SECRET=otro-secreto-temporal-de-32-caracteres `
  node:24-alpine sh -c "npm ci && MONGO_URL=mongodb://mongodb:27017/ea_proyecto npm run seed"
```

### En Linux / macOS

```bash
docker run --rm --network ea-proyecto_ea-net \
  -v "$PWD/../EA_Proyecto_Backend:/app" \
  -v /app/node_modules \
  -w /app \
  -e JWT_SECRET=secreto-temporal-de-32-caracteres-para-seed \
  -e JWT_REFRESH_SECRET=otro-secreto-temporal-de-32-caracteres \
  node:24-alpine sh -c "npm ci && MONGO_URL=mongodb://mongodb:27017/ea_proyecto npm run seed"
```

* Los secretos JWT se pasan únicamente porque la validación de configuración del backend los exige al importar módulos, aunque el script de seed no los utiliza.
* El script `seed` no va incluido dentro de la imagen de producción de la API porque depende de `tsx` (dependencia de desarrollo).
* Si necesitas vaciar y regenerar los datos desde cero, puedes ejecutar el script con `--reset` (`npm run seed -- --reset`) o borrar el volumen de Docker como se indica abajo.

---

## Datos y persistencia

MongoDB almacena sus datos en el volumen persistente de Docker `ea-proyecto_mongo-data`, montado en `/data/db`. Este volumen sobrevive a `docker compose down` y a las actualizaciones de imágenes (`docker compose pull && docker compose up -d`), por lo que tus datos no se pierden al reiniciar o actualizar el sistema.

Para eliminar por completo la base de datos y empezar de cero:

```bash
docker compose down -v
docker compose up -d
```

Después, vuelve a ejecutar el comando del **Seed** de la sección anterior.

---

## Configuración: Local vs. Producción

Las imágenes, puertos y red interna están definidos en `docker-compose.yml`. La configuración del entorno se gestiona en `.env`:

| Qué quieres cambiar | Variable en `.env` | Descripción |
|---|---|---|
| Nombre de la base de datos | `MONGO_DB` | Por defecto `ea_proyecto` |
| Orígenes permitidos (CORS) | `CORS_ORIGIN` | En local: `http://localhost:4200,http://localhost:4300` (o el origen público en producción) |
| Claves y expiración JWT | `JWT_SECRET`, `JWT_REFRESH_SECRET`, `JWT_EXPIRES_IN`, `JWT_REFRESH_EXPIRES_IN` | Secretos de firma y duración de los tokens |

### Actualizar la API
La API lee sus variables de entorno al arrancar. Si modificas `.env`, basta con ejecutar:

```bash
docker compose up -d
```

Docker Compose detectará los cambios en el entorno y recreará únicamente el contenedor `api` sin necesidad de reconstruir ninguna imagen.

### Actualizar los Frontends (Angular / React)
Las URLs de conexión hacia la API se compilan dentro de las imágenes estáticas de Nginx (`src/environments/environment.prod.ts` en Angular y `src/config/environment.ts` en React). Para apuntar a un servidor distinto (por ejemplo, una IP pública en producción):

1. Modifica la URL en el repositorio correspondiente (Angular o React) y haz `git push` a `main`.
2. Espera a que el workflow de GitHub Actions publique la nueva imagen en `ghcr.io`.
3. En este repositorio de despliegue, actualiza y levanta los contenedores:

```bash
docker compose pull
docker compose up -d
```