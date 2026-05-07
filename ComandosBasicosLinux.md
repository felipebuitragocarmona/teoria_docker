# Guía Básica de Consola Linux

La consola (o terminal) permite controlar Linux escribiendo comandos.
Piensa en ella como un explorador de archivos + herramientas, pero en texto.

Ingresar al siguiente enlace https://bellard.org/jslinux/

---

# 1. Saber dónde estás

```bash
pwd
```

Significa: **Print Working Directory**

Muestra la ruta actual.

Ejemplo:

```bash
/home/felipe
```

---

# 2. Ver archivos y carpetas

```bash
ls
```

Muestra el contenido de la carpeta actual.

Opciones útiles:

```bash
ls -l
```

Vista detallada.

```bash
ls -a
```

Muestra archivos ocultos.

---

# 3. Entrar a una carpeta

```bash
cd nombre_carpeta
```

Ejemplo:

```bash
cd documentos
```

---

# 4. Volver atrás

```bash
cd ..
```

Sube un nivel.

---

# 5. Ir a tu carpeta personal

```bash
cd ~
```

---

# 6. Crear carpetas

```bash
mkdir mi_carpeta
```

Ejemplo:

```bash
mkdir proyectos
```

Crear varias:

```bash
mkdir carpeta1 carpeta2 carpeta3
```

---

# 7. Eliminar carpetas

## Carpeta vacía

```bash
rmdir nombre_carpeta
```

## Carpeta con contenido

```bash
rm -r nombre_carpeta
```

⚠️ Mucho cuidado: elimina TODO dentro.

---

# 8. Crear archivos

## Archivo vacío

```bash
touch archivo.txt
```

---

# 9. Ver contenido de un archivo

```bash
cat archivo.txt
```

---

# 10. Editar archivos rápido

## Con nano

```bash
nano archivo.txt
```

Guardar:

* `CTRL + O`
* Enter

Salir:

* `CTRL + X`

---

# 11. Eliminar archivos

```bash
rm archivo.txt
```

---

# 12. Copiar archivos

```bash
cp archivo.txt copia.txt
```

---

# 13. Mover o renombrar archivos

```bash
mv archivo.txt nuevo.txt
```

Mover archivo:

```bash
mv archivo.txt /home/usuario/documentos
```

---

# 14. Limpiar la terminal

```bash
clear
```

Atajo:

```bash
CTRL + L
```

---

# 15. Ver la IP

## IP local

```bash
ip a
```

o

```bash
hostname -I
```

---

# 16. Ver información del sistema

```bash
uname -a
```

---

# 17. Ver procesos activos

```bash
top
```

Salir con:

```bash
q
```

---

# 18. Ver espacio en disco

```bash
df -h
```

---

# 19. Ver uso de memoria RAM

```bash
free -h
```

---

# 20. Descargar archivos desde internet

```bash
wget URL
```

Ejemplo:

```bash
wget https://archivo.com/file.zip
```

---

# 21. Permisos básicos

Dar permiso de ejecución:

```bash
chmod +x archivo.sh
```

---

# 22. Ejecutar scripts

```bash
./archivo.sh
```

---

# 23. Buscar archivos

```bash
find . -name archivo.txt
```

---

# 24. Historial de comandos

```bash
history
```

---

# 25. Apagar o reiniciar

## Apagar

```bash
sudo shutdown now
```

## Reiniciar

```bash
sudo reboot
```

---

# Conceptos importantes

| Símbolo | Significado      |
| ------- | ---------------- |
| `.`     | carpeta actual   |
| `..`    | carpeta anterior |
| `~`     | carpeta personal |
| `/`     | raíz del sistema |

