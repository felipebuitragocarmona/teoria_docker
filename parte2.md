# Tutorial: API REST Flask con Arquitectura en Capas y Docker

## 📋 Tabla de Contenidos
1. Introducción
2. Estructura del Proyecto
3. Capa de Acceso a Datos
4. Capa de Negocio
5. Capa de Vista (API)
6. Configuración de Docker
7. Pruebas y Ejecución

---

## 1. Introducción

En este tutorial aprenderás a crear una API REST con Flask utilizando una **arquitectura en capas** que separa responsabilidades y facilita el mantenimiento. Implementaremos un CRUD completo para gestionar usuarios y lo empaquetaremos en un contenedor Docker con **hot-reload** para desarrollo.

### ¿Qué vamos a construir?

- **Capa de Vista**: Endpoints REST (Flask)
- **Capa de Negocio**: Lógica de validación
- **Capa de Datos**: Persistencia en JSON
- **Docker**: Contenedor con volúmenes para desarrollo

---

## 2. Estructura del Proyecto

Primero, crea la siguiente estructura de carpetas:

```
flask-crud-app/
│
├── app/
│   ├── __init__.py
│   ├── routes/
│   │   ├── __init__.py
│   │   └── user_routes.py          # Capa de Vista
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   └── user_service.py         # Capa de Negocio
│   │
│   └── repositories/
│       ├── __init__.py
│       └── user_repository.py      # Capa de Datos
│
├── data/
│   └── users.json                  # Base de datos JSON
│
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── run.py                          # Punto de entrada
```

Crea todas estas carpetas y archivos vacíos por ahora.

---

## 3. Capa de Acceso a Datos

### 📄 `app/repositories/user_repository.py`

Esta capa se encarga de **leer y escribir** en el archivo JSON.

```python
import json
import os
from typing import List, Optional

class UserRepository:
    def __init__(self, db_path='data/users.json'):
        self.db_path = db_path
        self._ensure_db_exists()
    
    def _ensure_db_exists(self):
        """Crea el archivo JSON si no existe"""
        os.makedirs(os.path.dirname(self.db_path), exist_ok=True)
        if not os.path.exists(self.db_path):
            with open(self.db_path, 'w') as f:
                json.dump([], f)
    
    def _read_db(self) -> List[dict]:
        """Lee todos los usuarios del JSON"""
        with open(self.db_path, 'r') as f:
            return json.load(f)
    
    def _write_db(self, users: List[dict]):
        """Escribe usuarios en el JSON"""
        with open(self.db_path, 'w') as f:
            json.dump(users, f, indent=2)
    
    def get_all(self) -> List[dict]:
        """Obtiene todos los usuarios"""
        return self._read_db()
    
    def get_by_id(self, user_id: int) -> Optional[dict]:
        """Obtiene un usuario por ID"""
        users = self._read_db()
        return next((u for u in users if u['id'] == user_id), None)
    
    def create(self, user_data: dict) -> dict:
        """Crea un nuevo usuario"""
        users = self._read_db()
        
        # Generar ID automático
        new_id = max([u['id'] for u in users], default=0) + 1
        user_data['id'] = new_id
        
        users.append(user_data)
        self._write_db(users)
        return user_data
    
    def update(self, user_id: int, user_data: dict) -> Optional[dict]:
        """Actualiza un usuario existente"""
        users = self._read_db()
        
        for i, user in enumerate(users):
            if user['id'] == user_id:
                user_data['id'] = user_id
                users[i] = user_data
                self._write_db(users)
                return user_data
        
        return None
    
    def delete(self, user_id: int) -> bool:
        """Elimina un usuario"""
        users = self._read_db()
        initial_length = len(users)
        
        users = [u for u in users if u['id'] != user_id]
        
        if len(users) < initial_length:
            self._write_db(users)
            return True
        
        return False
```

---

## 4. Capa de Negocio

### 📄 `app/services/user_service.py`

Esta capa contiene la **lógica de negocio** y validaciones.

```python
from typing import List, Optional
from app.repositories.user_repository import UserRepository

class UserService:
    def __init__(self):
        self.repository = UserRepository()
    
    def get_all_users(self) -> List[dict]:
        """Obtiene todos los usuarios"""
        return self.repository.get_all()
    
    def get_user(self, user_id: int) -> Optional[dict]:
        """Obtiene un usuario por ID"""
        return self.repository.get_by_id(user_id)
    
    def create_user(self, data: dict) -> tuple:
        """
        Crea un usuario con validaciones
        Retorna: (usuario, error)
        """
        # Validaciones
        if not data.get('nombre'):
            return None, "El nombre es obligatorio"
        
        if not data.get('email'):
            return None, "El email es obligatorio"
        
        if '@' not in data.get('email', ''):
            return None, "El email no es válido"
        
        # Verificar email único
        existing_users = self.repository.get_all()
        if any(u['email'] == data['email'] for u in existing_users):
            return None, "El email ya está registrado"
        
        # Crear usuario
        user_data = {
            'nombre': data['nombre'],
            'email': data['email'],
            'edad': data.get('edad'),
            'activo': data.get('activo', True)
        }
        
        created_user = self.repository.create(user_data)
        return created_user, None
    
    def update_user(self, user_id: int, data: dict) -> tuple:
        """
        Actualiza un usuario con validaciones
        Retorna: (usuario, error)
        """
        # Verificar que existe
        existing_user = self.repository.get_by_id(user_id)
        if not existing_user:
            return None, "Usuario no encontrado"
        
        # Validaciones
        if 'email' in data and '@' not in data['email']:
            return None, "El email no es válido"
        
        # Verificar email único (excepto el mismo usuario)
        if 'email' in data:
            users = self.repository.get_all()
            if any(u['email'] == data['email'] and u['id'] != user_id for u in users):
                return None, "El email ya está registrado"
        
        # Actualizar campos
        updated_data = {
            'nombre': data.get('nombre', existing_user['nombre']),
            'email': data.get('email', existing_user['email']),
            'edad': data.get('edad', existing_user.get('edad')),
            'activo': data.get('activo', existing_user.get('activo', True))
        }
        
        updated_user = self.repository.update(user_id, updated_data)
        return updated_user, None
    
    def delete_user(self, user_id: int) -> tuple:
        """
        Elimina un usuario
        Retorna: (éxito, error)
        """
        success = self.repository.delete(user_id)
        if not success:
            return False, "Usuario no encontrado"
        return True, None
```

---

## 5. Capa de Vista (API)

### 📄 `app/routes/user_routes.py`

Esta capa expone los **endpoints REST**.

```python
from flask import Blueprint, request, jsonify
from app.services.user_service import UserService

user_bp = Blueprint('users', __name__)
user_service = UserService()

@user_bp.route('/users', methods=['GET'])
def get_users():
    """GET /api/users - Obtener todos los usuarios"""
    users = user_service.get_all_users()
    return jsonify(users), 200

@user_bp.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    """GET /api/users/:id - Obtener un usuario"""
    user = user_service.get_user(user_id)
    
    if not user:
        return jsonify({'error': 'Usuario no encontrado'}), 404
    
    return jsonify(user), 200

@user_bp.route('/users', methods=['POST'])
def create_user():
    """POST /api/users - Crear usuario"""
    data = request.get_json()
    
    if not data:
        return jsonify({'error': 'No se enviaron datos'}), 400
    
    user, error = user_service.create_user(data)
    
    if error:
        return jsonify({'error': error}), 400
    
    return jsonify(user), 201

@user_bp.route('/users/<int:user_id>', methods=['PUT'])
def update_user(user_id):
    """PUT /api/users/:id - Actualizar usuario"""
    data = request.get_json()
    
    if not data:
        return jsonify({'error': 'No se enviaron datos'}), 400
    
    user, error = user_service.update_user(user_id, data)
    
    if error:
        return jsonify({'error': error}), 400 if error != "Usuario no encontrado" else 404
    
    return jsonify(user), 200

@user_bp.route('/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    """DELETE /api/users/:id - Eliminar usuario"""
    success, error = user_service.delete_user(user_id)
    
    if error:
        return jsonify({'error': error}), 404
    
    return jsonify({'message': 'Usuario eliminado correctamente'}), 200
```

### 📄 `app/__init__.py`

Configuración de la aplicación Flask:

```python
from flask import Flask

def create_app():
    app = Flask(__name__)
    
    # Registrar blueprints
    from app.routes.user_routes import user_bp
    app.register_blueprint(user_bp, url_prefix='/api')
    
    # Ruta de bienvenida
    @app.route('/')
    def home():
        return {
            'message': 'API de Usuarios - Flask',
            'endpoints': {
                'GET /api/users': 'Obtener todos los usuarios',
                'GET /api/users/:id': 'Obtener un usuario',
                'POST /api/users': 'Crear usuario',
                'PUT /api/users/:id': 'Actualizar usuario',
                'DELETE /api/users/:id': 'Eliminar usuario'
            }
        }
    
    return app
```

### 📄 `run.py`

Punto de entrada de la aplicación:

```python
from app import create_app

app = create_app()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

---

## 6. Configuración de Docker

### 📄 `requirements.txt`

```
Flask==3.0.0
```

### 📄 `Dockerfile`

```dockerfile
FROM python:3.11-slim

# Establecer directorio de trabajo
WORKDIR /app

# Copiar dependencias
COPY requirements.txt .

# Instalar dependencias
RUN pip install --no-cache-dir -r requirements.txt

# Copiar código de la aplicación
COPY . .

# Exponer puerto
EXPOSE 5000

# Comando para ejecutar la app
CMD ["python", "run.py"]
```

### 📄 `docker-compose.yml`

Este archivo es **clave** para el desarrollo con hot-reload:

```yaml
version: '3.8'

services:
  flask-app:
    build: .
    container_name: flask-crud-api
    ports:
      - "5000:5000"
    volumes:
      # Monta el código fuente para hot-reload
      - ./app:/app/app
      - ./run.py:/app/run.py
      # Monta la carpeta de datos
      - ./data:/app/data
    environment:
      - FLASK_ENV=development
      - FLASK_DEBUG=1
    command: python run.py
```

**¿Qué hacen los volúmenes?**
- `./app:/app/app`: Sincroniza tu código local con el contenedor
- `./data:/app/data`: Persiste los datos JSON en tu máquina
- Cualquier cambio que hagas se refleja **automáticamente** sin reiniciar

---

## 7. Pruebas y Ejecución

### Paso 1: Construir y ejecutar

```bash
# Construir la imagen
docker-compose build

# Iniciar el contenedor
docker-compose up
```

### Paso 2: Probar los endpoints

**Crear un usuario:**
```bash
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "Juan Pérez",
    "email": "juan@example.com",
    "edad": 30
  }'
```

**Obtener todos los usuarios:**
```bash
curl http://localhost:5000/api/users
```

**Obtener un usuario específico:**
```bash
curl http://localhost:5000/api/users/1
```

**Actualizar un usuario:**
```bash
curl -X PUT http://localhost:5000/api/users/1 \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "Juan Carlos Pérez",
    "edad": 31
  }'
```

**Eliminar un usuario:**
```bash
curl -X DELETE http://localhost:5000/api/users/1
```

### Paso 3: Verificar hot-reload

1. Con el contenedor ejecutándose, edita `app/routes/user_routes.py`
2. Cambia algo (por ejemplo, un mensaje de respuesta)
3. **Guarda el archivo**
4. Haz una petición y verás el cambio **inmediatamente**

### Paso 4: Ver los datos persistidos

Abre el archivo `data/users.json` en tu editor. Verás algo como:

```json
[
  {
    "id": 1,
    "nombre": "Juan Pérez",
    "email": "juan@example.com",
    "edad": 30,
    "activo": true
  }
]
```

Puedes editar este archivo manualmente y los cambios se reflejan en la API.

---

## 🎯 Comandos Útiles de Docker

```bash
# Ver logs en tiempo real
docker-compose logs -f

# Detener el contenedor
docker-compose down

# Reconstruir después de cambios en Dockerfile
docker-compose up --build

# Acceder al contenedor
docker-compose exec flask-app bash

# Ver contenedores en ejecución
docker ps
```

---

## ✅ Resumen de la Arquitectura

```
Cliente (Postman/curl)
        ↓
[CAPA DE VISTA] user_routes.py
        ↓ (llama a)
[CAPA DE NEGOCIO] user_service.py
        ↓ (llama a)
[CAPA DE DATOS] user_repository.py
        ↓ (lee/escribe)
    users.json
```

**Ventajas:**
- ✅ Separación de responsabilidades
- ✅ Fácil de testear cada capa
- ✅ Escalable y mantenible
- ✅ Hot-reload con Docker
- ✅ Datos persistentes

---

## 🚀 Próximos Pasos

1. Agregar autenticación JWT
2. Implementar paginación en GET /users
3. Añadir filtros de búsqueda
4. Migrar a PostgreSQL
5. Agregar tests unitarios con pytest

¡Felicidades! Ya tienes una API REST profesional con Flask y Docker. 🎉
