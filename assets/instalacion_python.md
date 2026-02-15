 **1. Instalación de Python en Microsoft Windows**

 **1.1. Descarga del instalador**

1.  Acceder al sitio oficial de Python:  
    <https://www.python.org/downloads/> 
2.  Seleccionar el instalador adecuado (Windows installer 64‑bit es el recomendado para equipos modernos). 

 **1.2. Ejecución del instalador**

1.  Ejecutar el archivo descargado, por ejemplo: `python-3.x.x-amd64.exe`.
2.  En la ventana del instalador:
    *   Activar **"Add python.exe to PATH"** (crítico para usar Python en la terminal). 
    *   Activar **"Install launcher for all users"** si se requiere disponibilidad global.
3.  Seleccionar **"Install Now"**. El instalador desplegará archivos y configurará el intérprete. 

 **1.3. Verificación de la instalación**

1.  Abrir *Command Prompt* (`cmd`).
2.  Ejecutar:
    ```cmd
    python --version
    ```
    Si se muestra la versión instalada, el proceso fue exitoso. 

***

 **2. Instalación de Python en Ubuntu Linux**

Ubuntu incorpora Python en el sistema, pero suele ser necesario instalar o actualizar versiones específicas.

***

 **2.1. Instalación usando APT (método estándar)**

 **2.1.1. Actualizar repositorios**

```bash
sudo apt update
```


 **2.1.2. Instalar Python 3**

```bash
sudo apt install python3
```


 **2.1.3. Instalar pip**

```bash
sudo apt install python3-pip
```


 **2.1.4. Verificar instalación**

```bash
python3 --version
pip3 --version
```


***

 **2.2. Instalación de versiones más nuevas mediante PPA (Deadsnakes)**

Este método permite instalar versiones recientes como Python 3.13 o 3.14 sin reemplazar la versión del sistema.

 **2.2.1. Agregar repositorio**

```bash
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
```


 **2.2.2. Instalar nueva versión**

Ejemplo para Python 3.13:

```bash
sudo apt install python3.13
```


Verificar:

```bash
python3.13 --version
```


***

 **2.3. Compilación desde código fuente (instalación avanzada)**

 **2.3.1. Instalar dependencias**

```bash
sudo apt install build-essential libssl-dev libbz2-dev libreadline-dev libsqlite3-dev zlib1g-dev wget curl libffi-dev
```


 **2.3.2. Descargar y compilar código fuente**

```bash
wget https://www.python.org/ftp/python/<version>/Python-<version>.tgz
tar -xvf Python-<version>.tgz
cd Python-<version>
./configure --enable-optimizations
make -j $(nproc)
sudo make altinstall
```


***

 **3. Instalación de Python en Docker**

Instalar Python en Docker depende de si se usa una imagen oficial o se parte de una base mínima.

***

 **3.1. Usar la imagen oficial de Python (recomendado)**

La forma más limpia y estándar es usar las imágenes oficiales disponibles en Docker Hub.  
Ejemplo con Python 3.12.2:

```dockerfile
FROM python:3.12.2
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY .
CMD ["python", "app.py"]
```


 Notas:

*   Usar etiquetas específicas (**python:3.12.2**) evita cambios inesperados en actualizaciones.
*   Variantes disponibles: **slim**, **alpine**, **bookworm**, etc. para imágenes más ligeras. [\[hub.docker.com\]](https://hub.docker.com/_/python/)

***

 **3.2. Instalar Python manualmente en imágenes no‑Python**

Cuando se parte de `ubuntu:20.04` u otra base:

```dockerfile
FROM ubuntu:20.04
RUN apt-get update && apt-get install -y python3 python3-pip && rm -rf /var/lib/apt/lists/*
```


***

 **3.3. Buenas prácticas**

*   Evitar `sudo` en Dockerfile; trabajar siempre como root o cambiar contexto según necesidad. 
*   Mantener imágenes pequeñas usando `slim` o `alpine`.
*   Fijar versiones de paquetes y usar `requirements.txt`. [\[pytutorial.com\]](https://pytutorial.com/install-python-package-in-docker/)

***
