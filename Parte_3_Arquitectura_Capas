# Tutorial: CRUD de Usuarios con Docker (Arquitectura por Capas)

A continuación se muestra un proyecto siguiendo arquitectura por capas, para el CRUD de un usuario

## 📁 Estructura del Proyecto

```
usuario-crud/
├── src/                      ← 📦 TODO EL CÓDIGO AQUÍ
│   ├── app.py
│   ├── models/
│   │   └── usuario.py
│   ├── data_access/
│   │   └── usuario_dao.py
│   ├── business/
│   │   └── usuario_service.py
│   └── presentation/
│       └── usuario_routes.py
│
├── data/                     ← 💾 Persistencia
│   └── usuarios.json
│
├── requirements.txt          ← ⚙️ Configuración
├── Dockerfile                ← ⚙️ Configuración
├── docker-compose.yml        ← ⚙️ Configuración
├── .gitignore                ← ⚙️ Configuración
└── README.md                 ← 📖 Documentación
```
## 🚀 Pasos para ejecutar

### 1️⃣ Crea la estructura de carpetas

```bash
mkdir usuario-crud
cd usuario-crud
mkdir models data_access business presentation data
```

### 2️⃣ Crea todos los archivos

Copia cada archivo del artefacto a su ubicación correspondiente.

```python
# models/usuario.py
from dataclasses import dataclass
from typing import Optional

@dataclass
class Usuario:
    """Modelo de Usuario"""
    id: Optional[int] = None
    nombre: str = ""
    email: str = ""
    edad: int = 0
    
    def to_dict(self):
        """Convierte el usuario a diccionario"""
        return {
            "id": self.id,
            "nombre": self.nombre,
            "email": self.email,
            "edad": self.edad
        }
    
    @staticmethod
    def from_dict(data: dict):
        """Crea un usuario desde un diccionario"""
        return Usuario(
            id=data.get("id"),
            nombre=data.get("nombre", ""),
            email=data.get("email", ""),
            edad=data.get("edad", 0)
        )
```
Ahora el archivo data_access/usuario_dao.py

``` python
# data_access/usuario_dao.py
import json
import os
import sys
from typing import List, Optional

# Agregar el directorio raíz al path para imports
sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))

from models.usuario import Usuario

class UsuarioDAO:
    """Data Access Object para Usuario"""
    
    def __init__(self, archivo: str = "data/usuarios.json"):
        self.archivo = archivo
        self._inicializar_archivo()
    
    def _inicializar_archivo(self):
        """Crea el archivo JSON si no existe"""
        os.makedirs(os.path.dirname(self.archivo), exist_ok=True)
        if not os.path.exists(self.archivo):
            with open(self.archivo, 'w') as f:
                json.dump([], f)
    
    def _leer_datos(self) -> List[dict]:
        """Lee todos los datos del JSON"""
        with open(self.archivo, 'r') as f:
            return json.load(f)
    
    def _guardar_datos(self, datos: List[dict]):
        """Guarda todos los datos al JSON"""
        with open(self.archivo, 'w') as f:
            json.dump(datos, f, indent=2)
    
    def obtener_todos(self) -> List[Usuario]:
        """Obtiene todos los usuarios"""
        datos = self._leer_datos()
        return [Usuario.from_dict(d) for d in datos]
    
    def obtener_por_id(self, id: int) -> Optional[Usuario]:
        """Obtiene un usuario por ID"""
        datos = self._leer_datos()
        for d in datos:
            if d.get("id") == id:
                return Usuario.from_dict(d)
        return None
    
    def crear(self, usuario: Usuario) -> Usuario:
        """Crea un nuevo usuario"""
        datos = self._leer_datos()
        
        # Generar ID automático
        nuevo_id = max([d.get("id", 0) for d in datos], default=0) + 1
        usuario.id = nuevo_id
        
        datos.append(usuario.to_dict())
        self._guardar_datos(datos)
        return usuario
    
    def actualizar(self, id: int, usuario: Usuario) -> Optional[Usuario]:
        """Actualiza un usuario existente"""
        datos = self._leer_datos()
        
        for i, d in enumerate(datos):
            if d.get("id") == id:
                usuario.id = id
                datos[i] = usuario.to_dict()
                self._guardar_datos(datos)
                return usuario
        return None
    
    def eliminar(self, id: int) -> bool:
        """Elimina un usuario"""
        datos = self._leer_datos()
        datos_filtrados = [d for d in datos if d.get("id") != id]
        
        if len(datos) != len(datos_filtrados):
            self._guardar_datos(datos_filtrados)
            return True
        return False
```
Ahora el archivo business/usuario_service.py
```python
# business/usuario_service.py
import sys
import os
from typing import List, Optional

# Agregar el directorio raíz al path para imports
sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))

from models.usuario import Usuario
from data_access.usuario_dao import UsuarioDAO

class UsuarioService:
    """Lógica de negocio para Usuario"""
    
    def __init__(self):
        self.dao = UsuarioDAO()
    
    def listar_usuarios(self) -> List[Usuario]:
        """Lista todos los usuarios"""
        return self.dao.obtener_todos()
    
    def obtener_usuario(self, id: int) -> Optional[Usuario]:
        """Obtiene un usuario por ID"""
        return self.dao.obtener_por_id(id)
    
    def crear_usuario(self, nombre: str, email: str, edad: int) -> tuple[Usuario, str]:
        """
        Crea un nuevo usuario con validaciones
        Retorna: (usuario, mensaje_error)
        """
        # Validaciones de negocio
        if not nombre or len(nombre.strip()) == 0:
            return None, "El nombre es obligatorio"
        
        if not email or "@" not in email:
            return None, "Email inválido"
        
        if edad < 0 or edad > 150:
            return None, "Edad debe estar entre 0 y 150"
        
        # Verificar email duplicado
        usuarios = self.dao.obtener_todos()
        if any(u.email == email for u in usuarios):
            return None, "El email ya está registrado"
        
        # Crear usuario
        usuario = Usuario(nombre=nombre.strip(), email=email, edad=edad)
        return self.dao.crear(usuario), None
    
    def actualizar_usuario(self, id: int, nombre: str, email: str, edad: int) -> tuple[Optional[Usuario], str]:
        """Actualiza un usuario existente"""
        # Validaciones
        if not nombre or len(nombre.strip()) == 0:
            return None, "El nombre es obligatorio"
        
        if not email or "@" not in email:
            return None, "Email inválido"
        
        if edad < 0 or edad > 150:
            return None, "Edad debe estar entre 0 y 150"
        
        # Verificar que el usuario existe
        usuario_existente = self.dao.obtener_por_id(id)
        if not usuario_existente:
            return None, "Usuario no encontrado"
        
        # Verificar email duplicado (excepto el mismo usuario)
        usuarios = self.dao.obtener_todos()
        if any(u.email == email and u.id != id for u in usuarios):
            return None, "El email ya está registrado"
        
        # Actualizar
        usuario = Usuario(nombre=nombre.strip(), email=email, edad=edad)
        return self.dao.actualizar(id, usuario), None
    
    def eliminar_usuario(self, id: int) -> tuple[bool, str]:
        """Elimina un usuario"""
        if not self.dao.obtener_por_id(id):
            return False, "Usuario no encontrado"
        
        exito = self.dao.eliminar(id)
        return exito, None if exito else "Error al eliminar"
```
Ahora el archivo presentation/usuario_routes.py
```python
# presentation/usuario_routes.py
import sys
import os
from flask import Blueprint, request, jsonify

# Agregar el directorio raíz al path para imports
sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))

from business.usuario_service import UsuarioService

usuario_bp = Blueprint('usuario', __name__, url_prefix='/api/usuarios')
service = UsuarioService()

@usuario_bp.route('/', methods=['GET'])
def listar():
    """GET /api/usuarios - Lista todos los usuarios"""
    usuarios = service.listar_usuarios()
    return jsonify([u.to_dict() for u in usuarios]), 200

@usuario_bp.route('/<int:id>', methods=['GET'])
def obtener(id):
    """GET /api/usuarios/{id} - Obtiene un usuario"""
    usuario = service.obtener_usuario(id)
    if usuario:
        return jsonify(usuario.to_dict()), 200
    return jsonify({"error": "Usuario no encontrado"}), 404

@usuario_bp.route('/', methods=['POST'])
def crear():
    """POST /api/usuarios - Crea un usuario"""
    data = request.get_json()
    
    if not data:
        return jsonify({"error": "Datos inválidos"}), 400
    
    usuario, error = service.crear_usuario(
        nombre=data.get('nombre', ''),
        email=data.get('email', ''),
        edad=data.get('edad', 0)
    )
    
    if error:
        return jsonify({"error": error}), 400
    
    return jsonify(usuario.to_dict()), 201

@usuario_bp.route('/<int:id>', methods=['PUT'])
def actualizar(id):
    """PUT /api/usuarios/{id} - Actualiza un usuario"""
    data = request.get_json()
    
    if not data:
        return jsonify({"error": "Datos inválidos"}), 400
    
    usuario, error = service.actualizar_usuario(
        id=id,
        nombre=data.get('nombre', ''),
        email=data.get('email', ''),
        edad=data.get('edad', 0)
    )
    
    if error:
        return jsonify({"error": error}), 400 if usuario is None else 404
    
    return jsonify(usuario.to_dict()), 200

@usuario_bp.route('/<int:id>', methods=['DELETE'])
def eliminar(id):
    """DELETE /api/usuarios/{id} - Elimina un usuario"""
    exito, error = service.eliminar_usuario(id)
    
    if error:
        return jsonify({"error": error}), 404
    
    return jsonify({"mensaje": "Usuario eliminado correctamente"}), 200
```
Ahora el archivo app.py

````python
# app.py
import sys
import os
from flask import Flask, jsonify

# Agregar el directorio actual al path para imports
sys.path.insert(0, os.path.dirname(__file__))

from presentation.usuario_routes import usuario_bp

app = Flask(__name__)

# Registrar rutas
app.register_blueprint(usuario_bp)

@app.route('/')
def home():
    """Ruta raíz con información de la API"""
    return jsonify({
        "mensaje": "API de Usuarios - CRUD Simple",
        "endpoints": {
            "listar": "GET /api/usuarios",
            "obtener": "GET /api/usuarios/{id}",
            "crear": "POST /api/usuarios",
            "actualizar": "PUT /api/usuarios/{id}",
            "eliminar": "DELETE /api/usuarios/{id}"
        }
    })

@app.route('/health')
def health():
    """Health check"""
    return jsonify({"status": "ok"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
````

Después es necesario requeriments.txt
````txt
Flask==3.0.0
````
Ahora el Dockerfile

````python
# Dockerfile
# Usa la imagen oficial de Python 3.11 en versión "slim"
# Es liviana y adecuada para entornos de producción.
FROM python:3.11-slim

# Define el directorio de trabajo dentro del contenedor.
# Todo se ejecutará desde /app.
WORKDIR /app

# Copia el archivo de dependencias al contenedor.
# Esto permite que Docker aproveche la cache si no cambian las dependencias.
COPY requirements.txt .

# Instala todas las dependencias del proyecto.
# --no-cache-dir evita archivos temporales y mantiene la imagen ligera.
RUN pip install --no-cache-dir -r requirements.txt

# Copia únicamente la carpeta src/ a /app/src dentro del contenedor.
# Esto es útil para mantener la estructura del proyecto más organizada.
COPY src/ ./src/

# Crea un directorio llamado "data".
# Este se usa para almacenar archivos o base de datos interna.
RUN mkdir -p data

# Expone el puerto 5000 para que el contenedor pueda recibir tráfico externo.
EXPOSE 5000

# Ejecuta la aplicación desde la carpeta src/
# Esto arranca tu app Flask: python src/app.py
CMD ["python", "src/app.py"]

````
Y por último docker-compose.yml

````python
version: '3.8'    # Versión del estándar de Docker Compose

services:         # Sección donde se definen los servicios (contenedores)

  app:            # Nombre del servicio principal (tu aplicación Flask)
    build: .      # Construye la imagen usando el Dockerfile en el directorio actual

    container_name: usuario-crud-app   # Nombre personalizado del contenedor

    ports:
      - "5000:5000"   # Expone el puerto 5000 del contenedor al puerto 5000 del host
                       # Permite acceder a tu aplicación en http://localhost:5000

    volumes:
      # Monta la carpeta src del proyecto local dentro del contenedor en /app/src
      # Permite HOT RELOAD porque los cambios en el código se reflejan de inmediato
      - ./src:/app/src

      # Monta el directorio "data" para persistir información
      # Cualquier archivo generado por la app se guarda fuera del contenedor
      - ./data:/app/data

    environment:
      - FLASK_ENV=development   # Activa modo desarrollo en Flask
      - FLASK_DEBUG=1           # Habilita recarga automática y debugger

    restart: unless-stopped     # Reinicia el contenedor salvo que tú lo detengas manualmente
````
### 3️⃣ Construye y ejecuta con Docker

```bash
# Construir y levantar el contenedor
docker-compose up --build

# O en segundo plano
docker-compose up -d --build
```



## 📂 Persistencia de datos

Los datos se guardan en `data/usuarios.json` y persisten incluso si detienes el contenedor gracias al volumen:

```yaml
- ./data:/app/data
```

## 🔄 Hot-Reload funcionando

Edita cualquier archivo `.py` y los cambios se reflejarán automáticamente:

1. Abre `business/usuario_service.py`
2. Cambia algo (ej: edad máxima de 150 a 120)
3. Guarda
4. ¡Flask se recarga automáticamente! ✨

## 🛑 Detener y limpiar

```bash
# Detener
docker-compose down

# Detener y eliminar volúmenes
docker-compose down -v

# Ver logs
docker-compose logs -f
```

## 🎯 Arquitectura explicada

```
┌─────────────────┐
│  PRESENTACIÓN   │  usuario_routes.py (Flask endpoints)
└────────┬────────┘
         │
┌────────▼────────┐
│    NEGOCIO      │  usuario_service.py (Validaciones)
└────────┬────────┘
         │
┌────────▼────────┐
│  ACCESO DATOS   │  usuario_dao.py (Lee/escribe JSON)
└────────┬────────┘
         │
┌────────▼────────┐
│    MODELO       │  usuario.py (Estructura de datos)
└─────────────────┘
```
