# Flask + Python 2.7 + Docker Compose con Hot Reload

## Estructura

```bash
flask_python27/
├── app.py
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

---

## `app.py`

```python
# -*- coding: utf-8 -*-

from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Servidor Flask en Python 2.7 con Hot Reload"

@app.route("/saludo/<nombre>")
def saludo(nombre):
    return "Hola {}".format(nombre)

if __name__ == "__main__":
    app.run(
        host="0.0.0.0",
        port=5000,
        debug=True
    )
```

---

## `requirements.txt`

```txt
Flask==1.1.4
Werkzeug==1.0.1
Jinja2==2.11.3
MarkupSafe==1.1.1
itsdangerous==1.1.0
click==7.1.2
```

---

## `Dockerfile` comentado

```dockerfile
# Usamos la imagen oficial de Python 2.7
FROM python:2.7

# Definimos la carpeta de trabajo dentro del contenedor
WORKDIR /app

# Copiamos primero el archivo de dependencias
# Esto ayuda a que Docker use cache si requirements.txt no cambia
COPY requirements.txt .

# Instalamos las dependencias del proyecto
RUN pip install --no-cache-dir -r requirements.txt

# Copiamos el resto del código al contenedor
COPY . .

# Comando por defecto para iniciar la aplicación
CMD ["python", "app.py"]
```

---

## `docker-compose.yml` comentado

```yaml
services:

  # Servicio principal de Flask
  flask_app:

    # Construye la imagen usando el Dockerfile actual
    build: .

    # Nombre del contenedor
    container_name: flask_python27

    # Mapea el puerto 5000 del contenedor al puerto 5000 de tu máquina
    ports:
      - "5000:5000"

    # Monta la carpeta actual dentro del contenedor
    # Esto permite hot reload porque los cambios locales se ven en /app
    volumes:
      - .:/app

    # Carpeta de trabajo dentro del contenedor
    working_dir: /app

    # Variables de entorno para Flask en modo desarrollo
    environment:
      FLASK_ENV: development
      FLASK_DEBUG: 1

    # Ejecuta la app
    # debug=True en app.py activa el recargado automático
    command: python app.py
```

---

# Levantar el servidor

```bash
docker compose up --build
```

---

# Probar en navegador

```txt
http://localhost:5000
```

Respuesta:

```txt
Servidor Flask en Python 2.7 con Hot Reload
```

---

# Probar ruta dinámica

```txt
http://localhost:5000/saludo/Gerardo
```

Respuesta:

```txt
Hola Gerardo
```

---

# Probar Hot Reload

Edita `app.py`:

```python
return "Servidor actualizado con hot reload"
```

Guarda el archivo.

Flask detectará el cambio y reiniciará automáticamente el servidor.

No necesitas ejecutar otra vez:

```bash
docker compose up --build
```

---

# Cuándo reconstruir

Solo reconstruye si cambias:

* `Dockerfile`
* `requirements.txt`
* dependencias instaladas

```bash
docker compose up --build
```

Para cambios en `app.py`, no reconstruyas.
