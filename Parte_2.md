# Tutorial: API REST Flask Simple con JSON y Docker

## 📋 Tabla de Contenidos
1. Introducción
2. Estructura del Proyecto
3. Implementación de la API
4. Configuración de Docker
5. Pruebas y Ejecución

---

## 1. Introducción

En este tutorial aprenderás a crear una API REST con Flask de forma **simple y directa**, sin arquitectura en capas ni clases. Todo el código trabajará con **diccionarios de Python** y persistencia en **archivos JSON**.

### ¿Qué vamos a construir?

- API REST con Flask
- CRUD completo de usuarios
- Persistencia en JSON
- Docker con hot-reload
- Sin arquitectura compleja
- Sin clases (solo funciones y diccionarios)

---

## 2. Estructura del Proyecto

Crea la siguiente estructura simple:

```
flask-simple-api/
│
├── app.py                  # Toda la lógica de la API
├── data/
│   └── users.json          # Base de datos JSON
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

---

## 3. Implementación de la API

### 📄 `app.py`

Este archivo contiene **toda la lógica** de la aplicación:

```python
from flask import Flask, request, jsonify
import json
import os
from datetime import datetime

app = Flask(__name__)
app.config['JSON_AS_ASCII'] = False

# Ruta del archivo JSON
DB_PATH = 'data/users.json'

# ==================== FUNCIONES AUXILIARES ====================

def ensure_db_exists():
    """Crea el archivo JSON si no existe"""
    os.makedirs(os.path.dirname(DB_PATH), exist_ok=True)
    if not os.path.exists(DB_PATH):
        with open(DB_PATH, 'w', encoding='utf-8') as f:
            json.dump([], f)

def read_users():
    """Lee todos los usuarios del archivo JSON"""
    ensure_db_exists()
    with open(DB_PATH, 'r', encoding='utf-8') as f:
        return json.load(f)

def write_users(users):
    """Escribe usuarios en el archivo JSON"""
    with open(DB_PATH, 'w', encoding='utf-8') as f:
        json.dump(users, f, indent=2, ensure_ascii=False)

def get_next_id():
    """Genera el siguiente ID disponible"""
    users = read_users()
    if not users:
        return 1
    return max(user['id'] for user in users) + 1

def find_user_by_id(user_id):
    """Busca un usuario por ID"""
    users = read_users()
    for user in users:
        if user['id'] == user_id:
            return user
    return None

def find_user_by_email(email):
    """Busca un usuario por email"""
    users = read_users()
    for user in users:
        if user['email'] == email:
            return user
    return None

def validate_user_data(data, is_update=False):
    """
    Valida los datos del usuario
    Retorna: (es_valido, mensaje_error)
    """
    # Validar nombre
    if not is_update or 'nombre' in data:
        nombre = data.get('nombre', '').strip()
        if not nombre:
            return False, "El nombre es obligatorio"
        if len(nombre) < 2:
            return False, "El nombre debe tener al menos 2 caracteres"
        if len(nombre) > 100:
            return False, "El nombre no puede exceder 100 caracteres"
    
    # Validar email
    if not is_update or 'email' in data:
        email = data.get('email', '').strip()
        if not email:
            return False, "El email es obligatorio"
        if '@' not in email or '.' not in email.split('@')[1]:
            return False, "El email no tiene un formato válido"
    
    # Validar edad
    if 'edad' in data and data['edad'] is not None:
        edad = data.get('edad')
        if not isinstance(edad, int):
            return False, "La edad debe ser un número entero"
        if edad < 0:
            return False, "La edad no puede ser negativa"
        if edad > 150:
            return False, "La edad no puede ser mayor a 150"
    
    return True, None

# ==================== RUTAS DE LA API ====================

@app.route('/')
def home():
    """Ruta de bienvenida"""
    return {
        'message': 'API de Usuarios - Flask Simple',
        'version': '1.0.0',
        'endpoints': {
            'GET /api/users': 'Obtener todos los usuarios',
            'GET /api/users?activos=true': 'Obtener solo usuarios activos',
            'GET /api/users/<id>': 'Obtener un usuario por ID',
            'GET /api/users/email/<email>': 'Buscar usuario por email',
            'POST /api/users': 'Crear usuario',
            'PUT /api/users/<id>': 'Actualizar usuario',
            'PATCH /api/users/<id>/toggle': 'Alternar estado activo/inactivo',
            'DELETE /api/users/<id>': 'Eliminar usuario',
            'GET /api/users/stats': 'Obtener estadísticas'
        }
    }

@app.route('/api/users', methods=['GET'])
def get_users():
    """GET /api/users - Obtener todos los usuarios"""
    users = read_users()
    
    # Filtrar solo activos si se solicita
    only_active = request.args.get('activos', 'false').lower() == 'true'
    if only_active:
        users = [user for user in users if user.get('activo', True)]
    
    return jsonify({
        'total': len(users),
        'usuarios': users
    }), 200

@app.route('/api/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    """GET /api/users/<id> - Obtener un usuario"""
    user = find_user_by_id(user_id)
    
    if not user:
        return jsonify({'error': 'Usuario no encontrado'}), 404
    
    return jsonify(user), 200

@app.route('/api/users/email/<email>', methods=['GET'])
def get_user_by_email(email):
    """GET /api/users/email/<email> - Buscar usuario por email"""
    user = find_user_by_email(email)
    
    if not user:
        return jsonify({'error': 'Usuario no encontrado'}), 404
    
    return jsonify(user), 200

@app.route('/api/users', methods=['POST'])
def create_user():
    """POST /api/users - Crear usuario"""
    data = request.get_json()
    
    if not data:
        return jsonify({'error': 'No se enviaron datos'}), 400
    
    # Validar datos
    es_valido, error = validate_user_data(data)
    if not es_valido:
        return jsonify({'error': error}), 400
    
    # Verificar email único
    if find_user_by_email(data['email']):
        return jsonify({'error': 'El email ya está registrado'}), 400
    
    # Crear nuevo usuario
    nuevo_usuario = {
        'id': get_next_id(),
        'nombre': data['nombre'].strip(),
        'email': data['email'].strip().lower(),
        'edad': data.get('edad'),
        'activo': data.get('activo', True),
        'fecha_creacion': datetime.now().isoformat()
    }
    
    # Guardar en el archivo
    users = read_users()
    users.append(nuevo_usuario)
    write_users(users)
    
    return jsonify({
        'message': 'Usuario creado exitosamente',
        'usuario': nuevo_usuario
    }), 201

@app.route('/api/users/<int:user_id>', methods=['PUT'])
def update_user(user_id):
    """PUT /api/users/<id> - Actualizar usuario"""
    data = request.get_json()
    
    if not data:
        return jsonify({'error': 'No se enviaron datos'}), 400
    
    # Buscar usuario
    users = read_users()
    user_index = None
    for i, user in enumerate(users):
        if user['id'] == user_id:
            user_index = i
            break
    
    if user_index is None:
        return jsonify({'error': 'Usuario no encontrado'}), 404
    
    # Validar datos
    es_valido, error = validate_user_data(data, is_update=True)
    if not es_valido:
        return jsonify({'error': error}), 400
    
    # Verificar email único (si se está actualizando)
    if 'email' in data:
        existing_user = find_user_by_email(data['email'])
        if existing_user and existing_user['id'] != user_id:
            return jsonify({'error': 'El email ya está registrado por otro usuario'}), 400
    
    # Actualizar campos
    usuario_actual = users[user_index]
    if 'nombre' in data:
        usuario_actual['nombre'] = data['nombre'].strip()
    if 'email' in data:
        usuario_actual['email'] = data['email'].strip().lower()
    if 'edad' in data:
        usuario_actual['edad'] = data['edad']
    if 'activo' in data:
        usuario_actual['activo'] = data['activo']
    
    # Guardar cambios
    users[user_index] = usuario_actual
    write_users(users)
    
    return jsonify({
        'message': 'Usuario actualizado exitosamente',
        'usuario': usuario_actual
    }), 200

@app.route('/api/users/<int:user_id>/toggle', methods=['PATCH'])
def toggle_user_status(user_id):
    """PATCH /api/users/<id>/toggle - Alternar estado activo/inactivo"""
    users = read_users()
    user_index = None
    
    for i, user in enumerate(users):
        if user['id'] == user_id:
            user_index = i
            break
    
    if user_index is None:
        return jsonify({'error': 'Usuario no encontrado'}), 404
    
    # Cambiar estado
    users[user_index]['activo'] = not users[user_index].get('activo', True)
    write_users(users)
    
    estado = 'activado' if users[user_index]['activo'] else 'desactivado'
    
    return jsonify({
        'message': f'Usuario {estado} correctamente',
        'usuario': users[user_index]
    }), 200

@app.route('/api/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    """DELETE /api/users/<id> - Eliminar usuario"""
    users = read_users()
    usuarios_filtrados = [user for user in users if user['id'] != user_id]
    
    if len(usuarios_filtrados) == len(users):
        return jsonify({'error': 'Usuario no encontrado'}), 404
    
    write_users(usuarios_filtrados)
    
    return jsonify({
        'message': 'Usuario eliminado correctamente'
    }), 200

@app.route('/api/users/stats', methods=['GET'])
def get_stats():
    """GET /api/users/stats - Obtener estadísticas"""
    users = read_users()
    activos = [user for user in users if user.get('activo', True)]
    
    return jsonify({
        'total_usuarios': len(users),
        'usuarios_activos': len(activos),
        'usuarios_inactivos': len(users) - len(activos)
    }), 200

# ==================== MANEJO DE ERRORES ====================

@app.errorhandler(404)
def not_found(error):
    return jsonify({'error': 'Recurso no encontrado'}), 404

@app.errorhandler(500)
def internal_error(error):
    return jsonify({'error': 'Error interno del servidor'}), 500

@app.errorhandler(400)
def bad_request(error):
    return jsonify({'error': 'Solicitud incorrecta'}), 400

# ==================== EJECUTAR APLICACIÓN ====================

if __name__ == '__main__':
    ensure_db_exists()
    app.run(host='0.0.0.0', port=5000, debug=True)
```

---

## 4. Configuración de Docker

### 📄 `requirements.txt`

```
Flask==3.0.0
Werkzeug==3.0.1
```

### 📄 `Dockerfile`

```dockerfile

# Imagen base: una imagen oficial de Python 3.11 basada en Debian slim.
# Importante: "slim" reduce tamaño al eliminar paquetes no esenciales,
# pero puede requerir instalar dependencias del sistema si tu app las necesita.
FROM python:3.11-slim

# Establece el directorio de trabajo dentro del contenedor.
# Todas las instrucciones posteriores (COPY, RUN, CMD, etc.) se ejecutan
# relativas a /app. Mantener un WORKDIR claro mejora la legibilidad.
WORKDIR /app

# Copia el archivo de requisitos al contenedor (ruta relativa al contexto de build).
# Esto se hace antes de copiar todo el código para aprovechar la cache de Docker:
# si requirements.txt no cambia, la capa con pip install se reutiliza.
COPY requirements.txt .

# Instala las dependencias de Python listadas en requirements.txt.
# --no-cache-dir evita que pip almacene archivos descargados en /root/.cache,
# reduciendo el tamaño final de la imagen.
# Es buena práctica agrupar (RUN) instalaciones para minimizar capas.
RUN pip install --no-cache-dir -r requirements.txt

# Copia todo el contenido del contexto de construcción al contenedor.
# Atención: si no usas .dockerignore, estarás copiando archivos innecesarios
# (venv, .git, datos locales), lo que incrementa el contexto y la imagen.
COPY . .

# Señala que la aplicación escuchará en el puerto 5000.
# EXPOSE es metadata útil para documentación y para algunas herramientas,
# pero no publica el puerto por sí mismo (eso se hace con -p/--publish al correr).
EXPOSE 5000

# Comando por defecto cuando se arranca el contenedor.
# Aquí se usa JSON array para evitar el wrapper shell (`sh -c`) y que los
# signals (SIGTERM) lleguen correctamente al proceso Python.
CMD ["python", "app.py"]


```

### 📄 `docker-compose.yml`

```yaml
version: '3.8'
# Versión del formato de docker-compose.
# Especificarla garantiza compatibilidad de sintaxis y características del archivo.
# La versión 3.8 es estable y funciona muy bien para entornos locales y de producción ligera.

services:
  flask-app:
    # Nombre del servicio: define un contenedor basado en la app Flask.
    # Docker Compose puede ejecutar múltiples servicios (db, redis, backend, etc.); este es uno de ellos.

    build: .
    # Indica que debe construir la imagen usando el Dockerfile ubicado en el directorio actual (.)
    # Esto es útil cuando tienes un Dockerfile personalizado.
    # Si ya tuvieras una imagen construida, podrías usar "image: nombre:tag" en lugar de build.

    container_name: flask-simple-api
    # Nombre legible del contenedor.
    # Si no se especifica, Docker genera uno automáticamente.
    # Importante: usar nombres fijos puede causar conflictos si creas múltiples instancias.

    ports:
      - "5000:5000"
    # Publica el puerto 5000 del contenedor en el 5000 de tu máquina.
    # Formato: "HOST:CONTENEDOR"
    # Importante:
    # - Contenedor escucha en 5000 (desde EXPOSE o app)
    # - Host también 5000 → accedes desde: http://localhost:5000

    volumes:
      # Hot-reload del código
      - ./app.py:/app/app.py
      # Esto monta el archivo app.py del host sobre el del contenedor.
      # Efecto: cambios en tu máquina se reflejan inmediatamente dentro del contenedor.
      # Esto es IDEAL para desarrollo, pero NO debe usarse en producción.

      # Persistencia de datos
      - ./data:/app/data
      # Permite guardar datos en la carpeta ./data de tu máquina.
      # Si no montaras un volumen, los datos dentro del contenedor se perderían al eliminarlo.

    environment:
      - FLASK_ENV=development
      # FLASK_ENV indica el entorno de ejecución.
      # "development": activa modo debug, recarga automática y mensajes detallados de error.

      - FLASK_DEBUG=1
      # Fuerza debug activado. Útil si FLASK_ENV no es suficiente.
      # Jamás usar esto en producción (expondría stacktraces y ejecución remota de código).

      - PYTHONUNBUFFERED=1
      # Hace que Python escriba logs directamente en la salida estándar (sin buffer).
      # Es importante para que los logs se muestren inmediatamente en `docker logs`.

    command: python app.py
    # Sobrescribe el CMD del Dockerfile.
    # Le dice a Docker qué proceso debe ejecutarse dentro del contenedor.
    # Esto permite cambiar el comando sin modificar la imagen.
    # NOTA: si quisieras usar gunicorn en prod, aquí podrías poner:
    # command: gunicorn -b 0.0.0.0:5000 app:app

    restart: unless-stopped
    # Política de reinicio:
    # - Reinicia el contenedor automáticamente si se detiene por un error.
    # - No lo reinicia si tú lo detienes manualmente.
    # Es ideal para entornos pequeños o microservicios simples.
```

### 📄 `.dockerignore`

```
__pycache__
*.pyc
*.pyo
.Python
.git
.env
venv
```

---

## 5. Pruebas y Ejecución

### Paso 1: Construir y ejecutar

```bash
# Construir la imagen
docker-compose build

# Iniciar el contenedor
docker-compose up
```

Verás:
```
flask-simple-api  |  * Running on all addresses (0.0.0.0)
flask-simple-api  |  * Running on http://127.0.0.1:5000
```

### Paso 2: Probar la API

**1. Crear usuarios:**

```bash
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "Juan Pérez",
    "email": "juan@example.com",
    "edad": 30
  }'
```

Respuesta:
```json
{
  "message": "Usuario creado exitosamente",
  "usuario": {
    "id": 1,
    "nombre": "Juan Pérez",
    "email": "juan@example.com",
    "edad": 30,
    "activo": true,
    "fecha_creacion": "2024-01-15T10:30:00.123456"
  }
}
```

**2. Crear más usuarios:**

```bash
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "María García",
    "email": "maria@example.com",
    "edad": 25,
    "activo": true
  }'

curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "Carlos López",
    "email": "carlos@example.com",
    "edad": 35
  }'
```

**3. Obtener todos los usuarios:**

```bash
curl http://localhost:5000/api/users
```

Respuesta:
```json
{
  "total": 3,
  "usuarios": [
    {
      "id": 1,
      "nombre": "Juan Pérez",
      "email": "juan@example.com",
      "edad": 30,
      "activo": true,
      "fecha_creacion": "2024-01-15T10:30:00.123456"
    },
    ...
  ]
}
```

**4. Obtener solo usuarios activos:**

```bash
curl http://localhost:5000/api/users?activos=true
```

**5. Obtener un usuario específico:**

```bash
curl http://localhost:5000/api/users/1
```

**6. Buscar por email:**

```bash
curl http://localhost:5000/api/users/email/juan@example.com
```

**7. Actualizar usuario:**

```bash
curl -X PUT http://localhost:5000/api/users/1 \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "Juan Carlos Pérez",
    "edad": 31
  }'
```

**8. Alternar estado:**

```bash
curl -X PATCH http://localhost:5000/api/users/1/toggle
```

**9. Ver estadísticas:**

```bash
curl http://localhost:5000/api/users/stats
```

Respuesta:
```json
{
  "total_usuarios": 3,
  "usuarios_activos": 2,
  "usuarios_inactivos": 1
}
```

**10. Eliminar usuario:**

```bash
curl -X DELETE http://localhost:5000/api/users/1
```

### Paso 3: Probar validaciones

**Email inválido:**
```bash
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "Test",
    "email": "invalido"
  }'
```

Respuesta:
```json
{
  "error": "El email no tiene un formato válido"
}
```

**Nombre muy corto:**
```bash
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "A",
    "email": "test@example.com"
  }'
```

Respuesta:
```json
{
  "error": "El nombre debe tener al menos 2 caracteres"
}
```

**Email duplicado:**
```bash
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "Otro Usuario",
    "email": "maria@example.com"
  }'
```

Respuesta:
```json
{
  "error": "El email ya está registrado"
}
```

**Edad negativa:**
```bash
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "Test",
    "email": "test@example.com",
    "edad": -5
  }'
```

Respuesta:
```json
{
  "error": "La edad no puede ser negativa"
}
```

### Paso 4: Verificar hot-reload

1. Con el contenedor ejecutándose, abre `app.py`
2. Cambia el mensaje de bienvenida en la ruta `/`
3. Guarda el archivo
4. Visita `http://localhost:5000` y verás el cambio **inmediatamente**

### Paso 5: Ver datos persistidos

Abre `data/users.json`:

```json
[
  {
    "id": 1,
    "nombre": "Juan Pérez",
    "email": "juan@example.com",
    "edad": 30,
    "activo": true,
    "fecha_creacion": "2024-01-15T10:30:00.123456"
  },
  {
    "id": 2,
    "nombre": "María García",
    "email": "maria@example.com",
    "edad": 25,
    "activo": true,
    "fecha_creacion": "2024-01-15T10:35:00.654321"
  }
]
```

Puedes editar este archivo directamente y los cambios se reflejan en la API.

---

## 🎯 Comandos Útiles de Docker

```bash
# Ver logs
docker-compose logs -f

# Detener
docker-compose down

# Reiniciar
docker-compose restart

# Reconstruir
docker-compose up --build

# Acceder al contenedor
docker-compose exec flask-app bash

# Ver contenedores
docker ps

# Limpiar todo
docker-compose down -v
```

---

## 📊 Estructura Simple

```
Cliente (Postman/curl)
        ↓
    app.py
    ├── Rutas Flask
    ├── Funciones auxiliares
    ├── Validaciones
    └── Lectura/Escritura JSON
        ↓
    users.json
```

**Ventajas de este enfoque:**

✅ **Simplicidad**: Todo en un solo archivo  
✅ **Fácil de entender**: No hay abstracciones complejas  
✅ **Rápido de desarrollar**: Menos código, menos archivos  
✅ **Perfecto para proyectos pequeños**: No hay sobre-ingeniería  
✅ **Hot-reload**: Cambios instantáneos  
✅ **Persistencia**: Datos guardados en JSON  
