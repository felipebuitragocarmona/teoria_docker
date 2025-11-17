# 🚀 Tutorial: CRUD de Usuarios con Docker + MySQL (Arquitectura por Capas)

Proyecto microservicio gestión de usuarios usando Docker
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
├── init.sql                  ← 📄 Script inicial de base de datos
├── requirements.txt          ← ⚙️ Dependencias
├── Dockerfile                ← ⚙️ Imagen Docker
└── docker-compose.yml        ← ⚙️ Orquestación
```

---

## 📝 Archivos del Proyecto

### 1️⃣ `src/models/usuario.py`

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

### 2️⃣ `src/data_access/usuario_dao.py`
````python
# src/data_access/usuario_dao.py
import sys
import os
import mysql.connector
from typing import List, Optional

# Agregar el directorio raíz al path para imports
sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))

from models.usuario import Usuario

class UsuarioDAO:
    """Data Access Object para Usuario con MySQL"""
    
    def __init__(self):
        """Inicializa la conexión a MySQL"""
        self.config = {
            'host': os.getenv('DB_HOST', 'db'),
            'user': os.getenv('DB_USER', 'root'),
            'password': os.getenv('DB_PASSWORD', 'toor'),
            'database': os.getenv('DB_NAME', 'usuarios_db'),
            'port': int(os.getenv('DB_PORT', 3306))
        }
    
    def _get_connection(self):
        """Obtiene una conexión a la base de datos"""
        return mysql.connector.connect(**self.config)
    
    def obtener_todos(self) -> List[Usuario]:
        """Obtiene todos los usuarios"""
        conn = self._get_connection()
        cursor = conn.cursor(dictionary=True)
        
        try:
            cursor.execute("SELECT * FROM usuarios")
            resultados = cursor.fetchall()
            return [Usuario.from_dict(row) for row in resultados]
        finally:
            cursor.close()
            conn.close()
    
    def obtener_por_id(self, id: int) -> Optional[Usuario]:
        """Obtiene un usuario por ID"""
        conn = self._get_connection()
        cursor = conn.cursor(dictionary=True)
        
        try:
            cursor.execute("SELECT * FROM usuarios WHERE id = %s", (id,))
            resultado = cursor.fetchone()
            return Usuario.from_dict(resultado) if resultado else None
        finally:
            cursor.close()
            conn.close()
    
    def crear(self, usuario: Usuario) -> Usuario:
        """Crea un nuevo usuario"""
        conn = self._get_connection()
        cursor = conn.cursor()
        
        try:
            query = "INSERT INTO usuarios (nombre, email, edad) VALUES (%s, %s, %s)"
            cursor.execute(query, (usuario.nombre, usuario.email, usuario.edad))
            conn.commit()
            
            # Obtener el ID generado
            usuario.id = cursor.lastrowid
            return usuario
        finally:
            cursor.close()
            conn.close()
    
    def actualizar(self, id: int, usuario: Usuario) -> Optional[Usuario]:
        """Actualiza un usuario existente"""
        conn = self._get_connection()
        cursor = conn.cursor()
        
        try:
            query = "UPDATE usuarios SET nombre = %s, email = %s, edad = %s WHERE id = %s"
            cursor.execute(query, (usuario.nombre, usuario.email, usuario.edad, id))
            conn.commit()
            
            if cursor.rowcount > 0:
                usuario.id = id
                return usuario
            return None
        finally:
            cursor.close()
            conn.close()
    
    def eliminar(self, id: int) -> bool:
        """Elimina un usuario"""
        conn = self._get_connection()
        cursor = conn.cursor()
        
        try:
            cursor.execute("DELETE FROM usuarios WHERE id = %s", (id,))
            conn.commit()
            return cursor.rowcount > 0
        finally:
            cursor.close()
            conn.close()
    
    def email_existe(self, email: str, excluir_id: Optional[int] = None) -> bool:
        """Verifica si un email ya existe (excluyendo opcionalmente un ID)"""
        conn = self._get_connection()
        cursor = conn.cursor()
        
        try:
            if excluir_id:
                cursor.execute(
                    "SELECT COUNT(*) FROM usuarios WHERE email = %s AND id != %s",
                    (email, excluir_id)
                )
            else:
                cursor.execute("SELECT COUNT(*) FROM usuarios WHERE email = %s", (email,))
            
            count = cursor.fetchone()[0]
            return count > 0
        finally:
            cursor.close()
            conn.close()
````

### 3️⃣ `src/business/usuario_service.py`
````python
# src/business/usuario_service.py
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
        if self.dao.email_existe(email):
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
        if self.dao.email_existe(email, excluir_id=id):
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
````

### 4️⃣ `src/presentation/usuario_routes.py`
````python
# src/presentation/usuario_routes.py
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
````

### 5️⃣ `src/app.py`
````python
# src/app.py
import sys
import os
import time
import mysql.connector
from flask import Flask, jsonify

# Agregar el directorio actual al path para imports
sys.path.insert(0, os.path.dirname(__file__))

from presentation.usuario_routes import usuario_bp

app = Flask(__name__)

# Registrar rutas
app.register_blueprint(usuario_bp)

def esperar_db():
    """Espera a que MySQL esté disponible"""
    config = {
        'host': os.getenv('DB_HOST', 'db'),
        'user': os.getenv('DB_USER', 'root'),
        'password': os.getenv('DB_PASSWORD', 'toor'),
        'database': os.getenv('DB_NAME', 'usuarios_db'),
        'port': int(os.getenv('DB_PORT', 3306))
    }
    
    max_intentos = 30
    for intento in range(max_intentos):
        try:
            conn = mysql.connector.connect(**config)
            conn.close()
            print("✅ Conexión a MySQL establecida correctamente")
            return True
        except mysql.connector.Error as e:
            print(f"⏳ Esperando MySQL... intento {intento + 1}/{max_intentos}")
            time.sleep(2)
    
    print("❌ No se pudo conectar a MySQL")
    return False

@app.route('/')
def home():
    """Ruta raíz con información de la API"""
    return jsonify({
        "mensaje": "API de Usuarios - CRUD con MySQL",
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
    return jsonify({"status": "ok", "database": "mysql"}), 200

if __name__ == '__main__':
    # Esperar a que MySQL esté disponible
    esperar_db()
    
    # Iniciar Flask
    app.run(host='0.0.0.0', port=5000, debug=True)
````

### 6️⃣ `requirements.txt`
````python
Flask==3.0.0
mysql-connector-python==8.2.0
````

### 7️⃣ `init.sql` 
```python
-- init.sql
-- Script de inicialización de la base de datos

CREATE DATABASE IF NOT EXISTS usuarios_db;
USE usuarios_db;

CREATE TABLE IF NOT EXISTS usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    edad INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Datos de ejemplo (opcional)
INSERT INTO usuarios (nombre, email, edad) VALUES 
    ('Juan Pérez', 'juan@example.com', 30),
    ('María García', 'maria@example.com', 25),
    ('Carlos López', 'carlos@example.com', 35)
ON DUPLICATE KEY UPDATE nombre=nombre;
```

### 8️⃣ `Dockerfile`
````Docker
# Dockerfile

# Usa como base la imagen oficial de Python 3.11 en su versión "slim"
# (más ligera y adecuada para producción).
FROM python:3.11-slim

# Instala dependencias del sistema necesarias para compilar y usar MySQL (mysqlclient)
# - apt-get update: actualiza índices de paquetes
# - apt-get install:
#     default-libmysqlclient-dev → librerías de MySQL requeridas por mysqlclient
#     gcc → compilador necesario para compilar extensiones Python nativas
# - rm -rf /var/lib/apt/lists/* → limpia caché de apt para reducir tamaño de la imagen
RUN apt-get update && apt-get install -y \
    default-libmysqlclient-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Define /app como el directorio de trabajo dentro del contenedor.
# Todos los comandos posteriores se ejecutarán desde aquí.
WORKDIR /app

# Copia el archivo requirements.txt al contenedor.
# Esto permite aprovechar la cache de Docker si no cambia este archivo.
COPY requirements.txt .

# Instala las dependencias de Python listadas en requirements.txt
# --no-cache-dir evita almacenar archivos temporales, manteniendo la imagen ligera.
RUN pip install --no-cache-dir -r requirements.txt

# Copia todo el código fuente del proyecto desde la carpeta local src/
# hacia la carpeta interna /app/src del contenedor.
COPY src/ ./src/

# Declara que la aplicación usará el puerto 5000.
# Esto no abre el puerto, solo lo documenta para Docker.
EXPOSE 5000

# Comando por defecto que se ejecutará al iniciar el contenedor.
# Lanza la aplicación Flask ubicada en src/app.py
CMD ["python", "src/app.py"]

````

### 9️⃣ `docker-compose.yml` 
````docker
services:                              # Sección principal donde se definen todos los contenedores del proyecto

  # ------------------------------------------------------------------
  # SERVICIO 1: Aplicación Flask
  # ------------------------------------------------------------------
  app:                                 # Nombre del servicio (se usa como hostname dentro de Docker)
    build: .                           # Construye la imagen usando el Dockerfile del directorio actual
    container_name: usuario-crud-app   # Nombre del contenedor (para verlo fácilmente con "docker ps")

    ports:
      - "5000:5000"                    # Mapeo de puertos: HOST 5000 -> CONTENEDOR 5000
                                       # Permite acceder a Flask desde http://localhost:5000

    volumes:
      - ./src:/app/src                 # Monta la carpeta 'src' del host dentro del contenedor
                                       # Esto permite HOT RELOAD: cualquier cambio en el código se refleja sin reconstruir

    environment:                       # Variables de entorno que se pasan al contenedor
      - FLASK_ENV=development          # Activa el modo desarrollo en Flask (debugger, reloader)
      - FLASK_DEBUG=1                  # Fuerza el modo depuración
      - DB_HOST=db                     # Nombre del servicio DB (Docker lo resuelve como hostname)
      - DB_USER=root                   # Usuario para conectarse a MySQL
      - DB_PASSWORD=toor               # Contraseña del usuario
      - DB_NAME=usuarios_db            # Base de datos que la aplicación usará
      - DB_PORT=3306                   # Puerto INTERNO del contenedor MySQL (no del host)
                                       # La app nunca debe usar 3307, porque se conecta INTERNAMENTE por el servicio 'db'

    depends_on:                        # Define dependencia y orden de arranque
      db:
        condition: service_healthy     # Espera hasta que el healthcheck del servicio "db" confirme que está listo

    restart: unless-stopped            # Reinicia automáticamente el contenedor si se detiene por error
                                       # No reinicia si lo detienes manualmente

    networks:
      - app-network                    # Conecta este servicio a la red definida abajo
                                       # Permite que 'app' pueda comunicarse con 'db' por nombre: db:3306


  # ------------------------------------------------------------------
  # SERVICIO 2: Base de datos MySQL
  # ------------------------------------------------------------------
  db:                                  # Nombre del servicio MySQL (hostname interno)
    image: mysql:8.0                   # Imagen oficial de MySQL 8.0
    container_name: usuario-crud-db    # Nombre del contenedor de MySQL

    ports:
      - "3307:3306"                    # Mapeo: HOST 3307 -> CONTENEDOR 3306
                                       # Significa: MySQL escucha en 3306 internamente
                                       # pero se expone al host en 3307 para evitar conflictos

    environment:
      - MYSQL_ROOT_PASSWORD=toor       # Contraseña del usuario root de MySQL
      - MYSQL_DATABASE=usuarios_db     # Crea automáticamente esta base de datos al iniciar

    volumes:
      - mysql_data:/var/lib/mysql      # Persistencia de datos
                                       # Los datos NO se pierden aunque destruyas el contenedor
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
                                       # Script SQL que se ejecuta automáticamente al crear la BD por primera vez
                                       # Útil para tablas iniciales, inserts, configuraciones, etc.

    healthcheck:                       # Verifica periódicamente que MySQL esté realmente funcionando
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-ptoor"]
                                       # Comando para comprobar el estado del servidor MySQL
      interval: 5s                     # Ejecuta el test cada 5 segundos
      timeout: 3s                      # Si tarda más de 3 segundos, lo marca como fallido
      retries: 10                      # Si falla 10 veces seguidas, el servicio se considera unhealthy

    restart: unless-stopped            # Igual que en app: reinicia automáticamente salvo detención manual

    networks:
      - app-network                    # Se conecta a la misma red para permitir comunicación directa con 'app'


# ----------------------------------------------------------------------
# Definición de REDES
# ----------------------------------------------------------------------
networks:
  app-network:                         # Red compartida entre app y db
    driver: bridge                     # bridge = red privada interna de Docker, aislada del host


# ----------------------------------------------------------------------
# Definición de VOLUMENES PERSISTENTES
# ----------------------------------------------------------------------
volumes:
  mysql_data:                          # Volumen persistente para la base de datos
                                       # Guardará todos los archivos de MySQL aunque el contenedor se elimine
````

## 🚀 Pasos para Ejecutar

### 1️⃣ Crear la estructura de carpetas

```bash
mkdir usuario-crud
cd usuario-crud
mkdir src
mkdir src/models src/data_access src/business src/presentation
```

### 2️⃣ Crear todos los archivos

Copia cada archivo de los artefactos anteriores en su ubicación:

```
usuario-crud/
├── src/
│   ├── app.py
│   ├── models/
│   │   └── usuario.py
│   ├── data_access/
│   │   └── usuario_dao.py
│   ├── business/
│   │   └── usuario_service.py
│   └── presentation/
│       └── usuario_routes.py
├── init.sql
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

### 3️⃣ Levantar el proyecto

```bash
# Construir y levantar los contenedores
docker-compose up --build

# O en segundo plano
docker-compose up -d --build
```

### 4️⃣ Verificar que todo funciona

```bash
# Ver logs
docker-compose logs -f

# Verificar contenedores corriendo
docker ps
```

---

## 🧪 Probar la API

### Listar usuarios (debería mostrar los 3 de ejemplo)

```bash
curl http://localhost:5000/api/usuarios
```

### Crear un usuario

```bash
curl -X POST http://localhost:5000/api/usuarios \
  -H "Content-Type: application/json" \
  -d '{"nombre": "Pedro Sánchez", "email": "pedro@example.com", "edad": 28}'
```

### Obtener por ID

```bash
curl http://localhost:5000/api/usuarios/1
```

### Actualizar

```bash
curl -X PUT http://localhost:5000/api/usuarios/1 \
  -H "Content-Type: application/json" \
  -d '{"nombre": "Juan Actualizado", "email": "juan@example.com", "edad": 31}'
```

### Eliminar

```bash
curl -X DELETE http://localhost:5000/api/usuarios/1
```

---

## 🗄️ Acceder a MySQL directamente

```bash
# Conectarse al contenedor de MySQL
docker exec -it usuario-crud-db mysql -u root -ptoor

# Dentro de MySQL:
USE usuarios_db;
SELECT * FROM usuarios;
SHOW TABLES;
DESCRIBE usuarios;
```

---

## 📊 Arquitectura con MySQL

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
│  ACCESO DATOS   │  usuario_dao.py (Queries SQL)
└────────┬────────┘
         │
┌────────▼────────┐
│  MYSQL (Docker) │  Contenedor separado con persistencia
└─────────────────┘
```

---

## 🔄 Hot-Reload

Edita cualquier archivo en `src/` y Flask se recargará automáticamente:

```bash
# Los cambios se reflejan sin reiniciar el contenedor
- ./src:/app/src
```

---

## 💾 Persistencia de Datos

Los datos de MySQL se guardan en un volumen de Docker:

```yaml
volumes:
  - mysql_data:/var/lib/mysql
```

**Esto significa:**
- Los datos sobreviven aunque detengas los contenedores
- Para eliminar TODO: `docker-compose down -v`

---

## 🛑 Comandos Útiles

```bash
# Detener
docker-compose down

# Detener y eliminar datos
docker-compose down -v

# Ver logs de la app
docker-compose logs -f app

# Ver logs de MySQL
docker-compose logs -f db

# Reiniciar solo la app
docker-compose restart app

# Reconstruir imágenes
docker-compose build --no-cache
```

---

## 🎯 Diferencias Clave vs JSON

| Aspecto | JSON | MySQL |
|---------|------|-------|
| **Persistencia** | Archivo local | Base de datos real |
| **Concurrencia** | ❌ Problemas con escrituras simultáneas | ✅ Maneja múltiples conexiones |
| **Escalabilidad** | ❌ Limitado | ✅ Producción ready |
| **Queries** | ❌ Cargar todo en memoria | ✅ Queries optimizadas |
| **Transacciones** | ❌ No soporta | ✅ ACID compliant |
| **ID auto-increment** | Manual | ✅ Nativo |

---

