# Tutorial: API REST Flask con Arquitectura en Capas, Modelo Usuario y Docker

## 📋 Tabla de Contenidos
1. Introducción
2. Estructura del Proyecto
3. Capa de Modelo
4. Capa de Acceso a Datos
5. Capa de Controladores
6. Capa de Vista (API)
7. Configuración de Docker
8. Pruebas y Ejecución

---

## 1. Introducción

En este tutorial aprenderás a crear una API REST con Flask utilizando una **arquitectura en capas con modelos de datos** que incluye:

- **Modelo**: Clase Usuario con validaciones
- **Capa de Vista**: Endpoints REST (Flask)
- **Capa de Controladores**: Lógica de negocio y validación
- **Capa de Datos**: Persistencia en JSON
- **Docker**: Contenedor con hot-reload para desarrollo

---

## 2. Estructura del Proyecto

Crea la siguiente estructura de carpetas:

```
flask-crud-app/
│
├── app/
│   ├── __init__.py
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   └── user.py                 # Modelo Usuario
│   │
│   ├── routes/
│   │   ├── __init__.py
│   │   └── user_routes.py          # Capa de Vista
│   │
│   ├── controllers/
│   │   ├── __init__.py
│   │   └── user_controller.py      # Capa de Controladores
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

---

## 3. Capa de Modelo

### 📄 `app/models/user.py`

Esta clase representa la **entidad Usuario** con sus propiedades y métodos.

```python
from datetime import datetime
from typing import Optional

class User:
    """Modelo de Usuario"""
    
    def __init__(
        self,
        nombre: str,
        email: str,
        edad: Optional[int] = None,
        activo: bool = True,
        id: Optional[int] = None,
        fecha_creacion: Optional[str] = None
    ):
        self.id = id
        self.nombre = nombre
        self.email = email
        self.edad = edad
        self.activo = activo
        self.fecha_creacion = fecha_creacion or datetime.now().isoformat()
    
    def to_dict(self) -> dict:
        """Convierte el objeto Usuario a diccionario"""
        return {
            'id': self.id,
            'nombre': self.nombre,
            'email': self.email,
            'edad': self.edad,
            'activo': self.activo,
            'fecha_creacion': self.fecha_creacion
        }
    
    @classmethod
    def from_dict(cls, data: dict) -> 'User':
        """Crea un objeto Usuario desde un diccionario"""
        return cls(
            id=data.get('id'),
            nombre=data.get('nombre'),
            email=data.get('email'),
            edad=data.get('edad'),
            activo=data.get('activo', True),
            fecha_creacion=data.get('fecha_creacion')
        )
    
    def update_from_dict(self, data: dict):
        """Actualiza los atributos del usuario desde un diccionario"""
        if 'nombre' in data:
            self.nombre = data['nombre']
        if 'email' in data:
            self.email = data['email']
        if 'edad' in data:
            self.edad = data['edad']
        if 'activo' in data:
            self.activo = data['activo']
    
    def validate(self) -> tuple[bool, Optional[str]]:
        """
        Valida el usuario
        Retorna: (es_valido, mensaje_error)
        """
        if not self.nombre or not self.nombre.strip():
            return False, "El nombre es obligatorio"
        
        if len(self.nombre) < 2:
            return False, "El nombre debe tener al menos 2 caracteres"
        
        if len(self.nombre) > 100:
            return False, "El nombre no puede exceder 100 caracteres"
        
        if not self.email or not self.email.strip():
            return False, "El email es obligatorio"
        
        if '@' not in self.email or '.' not in self.email.split('@')[1]:
            return False, "El email no tiene un formato válido"
        
        if self.edad is not None:
            if not isinstance(self.edad, int):
                return False, "La edad debe ser un número entero"
            
            if self.edad < 0:
                return False, "La edad no puede ser negativa"
            
            if self.edad > 150:
                return False, "La edad no puede ser mayor a 150"
        
        return True, None
    
    def __repr__(self) -> str:
        return f"<User(id={self.id}, nombre='{self.nombre}', email='{self.email}')>"
    
    def __eq__(self, other) -> bool:
        if not isinstance(other, User):
            return False
        return self.id == other.id
```

### 📄 `app/models/__init__.py`

```python
from app.models.user import User

__all__ = ['User']
```

---

## 4. Capa de Acceso a Datos

### 📄 `app/repositories/user_repository.py`

Esta capa maneja la **persistencia** trabajando con objetos `User`.

```python
import json
import os
from typing import List, Optional
from app.models.user import User

class UserRepository:
    """Repositorio para gestionar la persistencia de usuarios"""
    
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
        """Lee el archivo JSON y retorna lista de diccionarios"""
        try:
            with open(self.db_path, 'r', encoding='utf-8') as f:
                return json.load(f)
        except json.JSONDecodeError:
            return []
    
    def _write_db(self, users_data: List[dict]):
        """Escribe lista de diccionarios en el archivo JSON"""
        with open(self.db_path, 'w', encoding='utf-8') as f:
            json.dump(users_data, f, indent=2, ensure_ascii=False)
    
    def get_all(self) -> List[User]:
        """Obtiene todos los usuarios como objetos User"""
        users_data = self._read_db()
        return [User.from_dict(data) for data in users_data]
    
    def get_by_id(self, user_id: int) -> Optional[User]:
        """Obtiene un usuario por ID como objeto User"""
        users_data = self._read_db()
        user_dict = next((u for u in users_data if u['id'] == user_id), None)
        
        if user_dict:
            return User.from_dict(user_dict)
        return None
    
    def get_by_email(self, email: str) -> Optional[User]:
        """Obtiene un usuario por email como objeto User"""
        users_data = self._read_db()
        user_dict = next((u for u in users_data if u['email'] == email), None)
        
        if user_dict:
            return User.from_dict(user_dict)
        return None
    
    def exists_email(self, email: str, exclude_id: Optional[int] = None) -> bool:
        """Verifica si un email ya existe (excluyendo opcionalmente un ID)"""
        users_data = self._read_db()
        
        for user in users_data:
            if user['email'] == email:
                if exclude_id is None or user['id'] != exclude_id:
                    return True
        
        return False
    
    def create(self, user: User) -> User:
        """Crea un nuevo usuario y retorna el objeto con ID asignado"""
        users_data = self._read_db()
        
        # Generar ID automático
        new_id = max([u['id'] for u in users_data], default=0) + 1
        user.id = new_id
        
        # Agregar a la lista y guardar
        users_data.append(user.to_dict())
        self._write_db(users_data)
        
        return user
    
    def update(self, user: User) -> Optional[User]:
        """Actualiza un usuario existente"""
        users_data = self._read_db()
        
        for i, user_dict in enumerate(users_data):
            if user_dict['id'] == user.id:
                users_data[i] = user.to_dict()
                self._write_db(users_data)
                return user
        
        return None
    
    def delete(self, user_id: int) -> bool:
        """Elimina un usuario por ID"""
        users_data = self._read_db()
        initial_length = len(users_data)
        
        users_data = [u for u in users_data if u['id'] != user_id]
        
        if len(users_data) < initial_length:
            self._write_db(users_data)
            return True
        
        return False
    
    def count(self) -> int:
        """Cuenta el total de usuarios"""
        users_data = self._read_db()
        return len(users_data)
```

### 📄 `app/repositories/__init__.py`

```python
from app.repositories.user_repository import UserRepository

__all__ = ['UserRepository']
```

---

## 5. Capa de Controladores

### 📄 `app/controllers/user_controller.py`

Esta capa implementa la **lógica de negocio** trabajando con objetos `User`.

```python
from typing import List, Optional, Tuple
from app.models.user import User
from app.repositories.user_repository import UserRepository

class UserController:
    """Controlador para la lógica de negocio de usuarios"""
    
    def __init__(self):
        self.repository = UserRepository()
    
    def get_all_users(self) -> List[User]:
        """Obtiene todos los usuarios"""
        return self.repository.get_all()
    
    def get_user_by_id(self, user_id: int) -> Optional[User]:
        """Obtiene un usuario por ID"""
        return self.repository.get_by_id(user_id)
    
    def get_user_by_email(self, email: str) -> Optional[User]:
        """Obtiene un usuario por email"""
        return self.repository.get_by_email(email)
    
    def create_user(self, data: dict) -> Tuple[Optional[User], Optional[str]]:
        """
        Crea un nuevo usuario con validaciones
        Retorna: (usuario, mensaje_error)
        """
        try:
            # Crear objeto User desde el diccionario
            user = User(
                nombre=data.get('nombre', ''),
                email=data.get('email', ''),
                edad=data.get('edad'),
                activo=data.get('activo', True)
            )
            
            # Validar el modelo
            es_valido, error = user.validate()
            if not es_valido:
                return None, error
            
            # Verificar email único
            if self.repository.exists_email(user.email):
                return None, "El email ya está registrado"
            
            # Crear usuario en la base de datos
            created_user = self.repository.create(user)
            return created_user, None
            
        except Exception as e:
            return None, f"Error al crear usuario: {str(e)}"
    
    def update_user(self, user_id: int, data: dict) -> Tuple[Optional[User], Optional[str]]:
        """
        Actualiza un usuario existente con validaciones
        Retorna: (usuario, mensaje_error)
        """
        try:
            # Verificar que el usuario existe
            existing_user = self.repository.get_by_id(user_id)
            if not existing_user:
                return None, "Usuario no encontrado"
            
            # Actualizar campos del usuario existente
            existing_user.update_from_dict(data)
            
            # Validar el modelo actualizado
            es_valido, error = existing_user.validate()
            if not es_valido:
                return None, error
            
            # Verificar email único (excepto el mismo usuario)
            if self.repository.exists_email(existing_user.email, exclude_id=user_id):
                return None, "El email ya está registrado por otro usuario"
            
            # Actualizar en la base de datos
            updated_user = self.repository.update(existing_user)
            return updated_user, None
            
        except Exception as e:
            return None, f"Error al actualizar usuario: {str(e)}"
    
    def delete_user(self, user_id: int) -> Tuple[bool, Optional[str]]:
        """
        Elimina un usuario
        Retorna: (éxito, mensaje_error)
        """
        try:
            # Verificar que el usuario existe
            existing_user = self.repository.get_by_id(user_id)
            if not existing_user:
                return False, "Usuario no encontrado"
            
            # Eliminar usuario
            success = self.repository.delete(user_id)
            return success, None
            
        except Exception as e:
            return False, f"Error al eliminar usuario: {str(e)}"
    
    def get_active_users(self) -> List[User]:
        """Obtiene todos los usuarios activos"""
        all_users = self.repository.get_all()
        return [user for user in all_users if user.activo]
    
    def get_users_count(self) -> int:
        """Obtiene el total de usuarios"""
        return self.repository.count()
    
    def toggle_user_status(self, user_id: int) -> Tuple[Optional[User], Optional[str]]:
        """
        Alterna el estado activo/inactivo de un usuario
        Retorna: (usuario, mensaje_error)
        """
        try:
            user = self.repository.get_by_id(user_id)
            if not user:
                return None, "Usuario no encontrado"
            
            # Cambiar estado
            user.activo = not user.activo
            
            # Actualizar en la base de datos
            updated_user = self.repository.update(user)
            return updated_user, None
            
        except Exception as e:
            return None, f"Error al cambiar estado: {str(e)}"
```

### 📄 `app/controllers/__init__.py`

```python
from app.controllers.user_controller import UserController

__all__ = ['UserController']
```

---

## 6. Capa de Vista (API)

### 📄 `app/routes/user_routes.py`

Esta capa expone los **endpoints REST**.

```python
from flask import Blueprint, request, jsonify
from app.controllers.user_controller import UserController

user_bp = Blueprint('users', __name__)
user_controller = UserController()

@user_bp.route('/users', methods=['GET'])
def get_users():
    """GET /api/users - Obtener todos los usuarios"""
    # Parámetro opcional para filtrar solo activos
    only_active = request.args.get('activos', 'false').lower() == 'true'
    
    if only_active:
        users = user_controller.get_active_users()
    else:
        users = user_controller.get_all_users()
    
    # Convertir objetos User a diccionarios
    users_dict = [user.to_dict() for user in users]
    
    return jsonify({
        'total': len(users_dict),
        'usuarios': users_dict
    }), 200

@user_bp.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    """GET /api/users/:id - Obtener un usuario"""
    user = user_controller.get_user_by_id(user_id)
    
    if not user:
        return jsonify({'error': 'Usuario no encontrado'}), 404
    
    return jsonify(user.to_dict()), 200

@user_bp.route('/users/email/<email>', methods=['GET'])
def get_user_by_email(email):
    """GET /api/users/email/:email - Buscar usuario por email"""
    user = user_controller.get_user_by_email(email)
    
    if not user:
        return jsonify({'error': 'Usuario no encontrado'}), 404
    
    return jsonify(user.to_dict()), 200

@user_bp.route('/users', methods=['POST'])
def create_user():
    """POST /api/users - Crear usuario"""
    data = request.get_json()
    
    if not data:
        return jsonify({'error': 'No se enviaron datos'}), 400
    
    user, error = user_controller.create_user(data)
    
    if error:
        return jsonify({'error': error}), 400
    
    return jsonify({
        'message': 'Usuario creado exitosamente',
        'usuario': user.to_dict()
    }), 201

@user_bp.route('/users/<int:user_id>', methods=['PUT'])
def update_user(user_id):
    """PUT /api/users/:id - Actualizar usuario"""
    data = request.get_json()
    
    if not data:
        return jsonify({'error': 'No se enviaron datos'}), 400
    
    user, error = user_controller.update_user(user_id, data)
    
    if error:
        status_code = 404 if error == "Usuario no encontrado" else 400
        return jsonify({'error': error}), status_code
    
    return jsonify({
        'message': 'Usuario actualizado exitosamente',
        'usuario': user.to_dict()
    }), 200

@user_bp.route('/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    """DELETE /api/users/:id - Eliminar usuario"""
    success, error = user_controller.delete_user(user_id)
    
    if error:
        return jsonify({'error': error}), 404
    
    return jsonify({
        'message': 'Usuario eliminado correctamente'
    }), 200

@user_bp.route('/users/<int:user_id>/toggle', methods=['PATCH'])
def toggle_user_status(user_id):
    """PATCH /api/users/:id/toggle - Alternar estado activo/inactivo"""
    user, error = user_controller.toggle_user_status(user_id)
    
    if error:
        return jsonify({'error': error}), 404
    
    estado = 'activado' if user.activo else 'desactivado'
    return jsonify({
        'message': f'Usuario {estado} correctamente',
        'usuario': user.to_dict()
    }), 200

@user_bp.route('/users/stats', methods=['GET'])
def get_stats():
    """GET /api/users/stats - Obtener estadísticas"""
    all_users = user_controller.get_all_users()
    active_users = user_controller.get_active_users()
    
    return jsonify({
        'total_usuarios': len(all_users),
        'usuarios_activos': len(active_users),
        'usuarios_inactivos': len(all_users) - len(active_users)
    }), 200
```

### 📄 `app/routes/__init__.py`

```python
from app.routes.user_routes import user_bp

__all__ = ['user_bp']
```

### 📄 `app/__init__.py`

Configuración de la aplicación Flask:

```python
from flask import Flask

def create_app():
    app = Flask(__name__)
    
    # Configuración
    app.config['JSON_AS_ASCII'] = False  # Para caracteres especiales
    
    # Registrar blueprints
    from app.routes.user_routes import user_bp
    app.register_blueprint(user_bp, url_prefix='/api')
    
    # Ruta de bienvenida
    @app.route('/')
    def home():
        return {
            'message': 'API de Usuarios - Flask con Arquitectura en Capas',
            'version': '1.0.0',
            'endpoints': {
                'GET /api/users': 'Obtener todos los usuarios',
                'GET /api/users?activos=true': 'Obtener solo usuarios activos',
                'GET /api/users/:id': 'Obtener un usuario por ID',
                'GET /api/users/email/:email': 'Buscar usuario por email',
                'POST /api/users': 'Crear usuario',
                'PUT /api/users/:id': 'Actualizar usuario',
                'DELETE /api/users/:id': 'Eliminar usuario',
                'PATCH /api/users/:id/toggle': 'Alternar estado activo/inactivo',
                'GET /api/users/stats': 'Obtener estadísticas'
            }
        }
    
    # Manejador de errores 404
    @app.errorhandler(404)
    def not_found(error):
        return {'error': 'Recurso no encontrado'}, 404
    
    # Manejador de errores 500
    @app.errorhandler(500)
    def internal_error(error):
        return {'error': 'Error interno del servidor'}, 500
    
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

## 7. Configuración de Docker

### 📄 `requirements.txt`

```
Flask==3.0.0
Werkzeug==3.0.1
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
      - PYTHONUNBUFFERED=1
    command: python run.py
    restart: unless-stopped
```

### 📄 `.dockerignore`

Crea este archivo para excluir archivos innecesarios:

```
__pycache__
*.pyc
*.pyo
*.pyd
.Python
*.so
*.egg
*.egg-info
dist
build
.git
.gitignore
.env
venv
env
```

---

## 8. Pruebas y Ejecución

### Paso 1: Construir y ejecutar

```bash
# Construir la imagen
docker-compose build

# Iniciar el contenedor
docker-compose up
```

Deberías ver:
```
flask-crud-api  |  * Serving Flask app 'app'
flask-crud-api  |  * Debug mode: on
flask-crud-api  |  * Running on all addresses (0.0.0.0)
flask-crud-api  |  * Running on http://127.0.0.1:5000
```

### Paso 2: Probar los endpoints

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

```bash
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "María García",
    "email": "maria@example.com",
    "edad": 25,
    "activo": true
  }'
```

**2. Obtener todos los usuarios:**
```bash
curl http://localhost:5000/api/users
```

**3. Obtener solo usuarios activos:**
```bash
curl http://localhost:5000/api/users?activos=true
```

**4. Obtener un usuario específico:**
```bash
curl http://localhost:5000/api/users/1
```

**5. Buscar usuario por email:**
```bash
curl http://localhost:5000/api/users/email/juan@example.com
```

**6. Actualizar un usuario:**
```bash
curl -X PUT http://localhost:5000/api/users/1 \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "Juan Carlos Pérez",
    "edad": 31
  }'
```

**7. Alternar estado activo/inactivo:**
```bash
curl -X PATCH http://localhost:5000/api/users/1/toggle
```

**8. Obtener estadísticas:**
```bash
curl http://localhost:5000/api/users/stats
```

**9. Eliminar un usuario:**
```bash
curl -X DELETE http://localhost:5000/api/users/1
```

### Paso 3: Verificar hot-reload

1. Con el contenedor ejecutándose, edita `app/models/user.py`
2. Por ejemplo, cambia el límite de edad de 150 a 120
3. **Guarda el archivo**
4. Crea un usuario con edad 130 y verás el nuevo error **inmediatamente**

### Paso 4: Ver los datos persistidos

Abre el archivo `data/users.json` en tu editor:

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

### Paso 5: Probar validaciones

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

**Email duplicado:**
```bash
curl -X POST http://localhost:5000/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "nombre": "Otro Usuario",
    "email": "juan@example.com"
  }'
```

Respuesta:
```json
{
  "error": "El email ya está registrado"
}
```

---

## 🎯 Comandos Útiles de Docker

```bash
# Ver logs en tiempo real
docker-compose logs -f

# Ver logs solo de errores
docker-compose logs -f | grep ERROR

# Detener el contenedor
docker-compose down

# Reconstruir después de cambios en Dockerfile o requirements.txt
docker-compose up --build

# Acceder al contenedor
docker-compose exec flask-app bash

# Ver contenedores en ejecución
docker ps

# Reiniciar el contenedor
docker-compose restart

# Ver uso de recursos
docker stats flask-crud-api
```

---

## ✅ Resumen de la Arquitectura

```
Cliente (Postman/curl/Frontend)
            ↓
┌───────────────────────────────────┐
│   CAPA DE VISTA (Routes)          │
│   user_routes.py                  │
│   - Endpoints REST                │
│   - Validación de requests        │
│   - Serialización JSON            │
└───────────────┬───────────────────┘
                ↓
┌───────────────────────────────────┐
│   CAPA DE CONTROLADORES           │
│   user_controller.py              │
│   - Lógica de negocio             │
│   - Validaciones complejas        │
│   - Orquestación                  │
└───────────────┬───────────────────┘
                ↓
┌───────────────────────────────────┐
│   CAPA DE REPOSITORIO             │
│   user_repository.py              │
│   - Acceso a datos                │
│   - CRUD básico                   │
│   - Persistencia                  │
└───────────────┬───────────────────┘
                ↓
┌───────────────────────────────────┐
│   CAPA DE MODELO                  │
│   user.py                         │
│   - Entidad Usuario               │
│   - Validaciones básicas          │
│   - Serialización                 │
└───────────────┬───────────────────┘
                ↓
            users.json
```

**Ventajas de esta arquitectura:**

✅ **Separación de responsabilidades**: Cada capa tiene un propósito claro  
✅ **Facilita testing**: Puedes probar cada capa de forma independiente  
✅ **Escalabilidad**: Fácil agregar nuevas funcionalidades  
✅ **Mantenibilidad**: Código organizado y fácil de entender  
✅ **Reutilización**: El modelo User puede usarse en diferentes contextos  
✅ **Hot-reload**: Cambios instantáneos sin reiniciar Docker  

