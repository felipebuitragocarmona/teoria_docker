# Tutorial Plantilla Angular 14 y Node 14

## 1. Estructura esperada

Primero es necesario descargar la plantilla con el siguiente comando:

```
git clone https://github.com/akveo/ngx-admin.git test_nebular
```
Luego internamente dentro de esta se debe de crear 2 archivos, Dockerfile y docker-compose.yml
```bash
package.json
package-lock.json
angular.json
src/
Dockerfile
docker-compose.yml
```

---

## 2. Dockerfile comentado

Crea un archivo llamado `Dockerfile`:

```dockerfile
# Node compatible con ngx-admin y node-sass
FROM node:14-bullseye

# Carpeta de trabajo dentro del contenedor
WORKDIR /app

# Instalamos Angular CLI global
RUN npm install -g @angular/cli@14.2.4

# Copiamos package.json y package-lock.json
COPY package*.json ./

# Instalamos dependencias del proyecto
# --legacy-peer-deps ayuda con proyectos legacy
RUN npm install --legacy-peer-deps

# Copiamos el resto del proyecto
COPY . .

# Puerto Angular
EXPOSE 4200

# Levanta Angular con hot reload
CMD ["ng", "serve", "--host", "0.0.0.0", "--poll", "2000"]
```

---

## 3. docker-compose.yml comentado

Crea `docker-compose.yml`:

```yaml
services:

  ngx_admin:

    build: .

    container_name: ngx_admin_app

    ports:
      - "4200:4200"

    volumes:
      - .:/app
      - /app/node_modules

    working_dir: /app
```

---

## 4. Levantar la plantilla

```bash
docker compose up --build
```

Abre:

```txt
http://localhost:4200
```

---

## 5. Hot reload

Edita cualquier archivo, por ejemplo:

```txt
src/app/app.component.html
```

Guarda el cambio.

Angular debería recompilar automáticamente y refrescar la app.

---

## 6. Entrar al contenedor

```bash
docker compose exec nebular_app bash
```

Dentro puedes revisar:

```bash
node -v
npm -v
npx ng version
```

---

## 8. Comandos útiles

Levantar:

```bash
docker compose up
```

Reconstruir:

```bash
docker compose up --build
```

Detener:

```bash
docker compose down
```

Limpiar dependencias del contenedor:

```bash
docker compose down -v
docker compose up --build
```