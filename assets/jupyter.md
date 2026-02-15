#### Jupyter 

Es un entorno de desarrollo interactivo (IDE) diseñado para facilitar la computación científica, el análisis de datos y la creación de documentos computacionales. Se basa en el concepto de notebook, un tipo de documento que integra código ejecutable, visualizaciones, resultados, texto en formato enriquecido, ecuaciones y elementos interactivos en un mismo espacio de trabajo. Esta estructura permite prototipar, documentar y comunicar procesos de análisis de manera clara y reproducible, convirtiendo a Jupyter en una herramienta ampliamente utilizada en ciencia de datos, educación, investigación y desarrollo de software. 

El ecosistema de Jupyter incluye interfaces como Jupyter Notebook, orientada a una experiencia sencilla y centrada en la escritura secuencial, y JupyterLab, una plataforma más robusta que proporciona una interfaz modular con pestañas, terminales, editores de texto, explorador de archivos y múltiples paneles. Ambas herramientas permiten ejecutar código en diferentes lenguajes mediante kernels, aunque Python es el más popular. 

En cuanto a su modelo de licenciamiento, Jupyter es completamente software libre y de código abierto, distribuido bajo la licencia BSD de 3 cláusulas. Esta licencia es de tipo permisivo, lo que permite el uso, modificación, redistribución y uso comercial del software, siempre que se respeten ciertos requisitos mínimos: mantener el aviso de copyright, incluir el texto de la licencia y evitar el uso del nombre del proyecto o sus contribuidores para promocionar derivados sin permiso explícito. 

El licenciamiento BSD‑3 favorece la adopción amplia en entornos académicos, empresariales y comerciales debido a su flexibilidad y baja restricción. Además, Jupyter mantiene un modelo de copyright compartido, en el cual cada contribuidor conserva los derechos sobre sus aportaciones, pero el proyecto se distribuye colectivamente bajo la misma licencia. 
Jupyter, como proyecto, se compromete a permanecer 100% abierto y libre, garantizando que sus herramientas continuarán disponibles sin restricciones de acceso, alineadas con su misión de promover la computación interactiva en beneficio global.


**1. Instalación de Jupyter en Microsoft Windows**

Existen múltiples métodos para instalar Jupyter en Windows. Aquí se documentan los **dos métodos oficiales más utilizados: instalación con pip y con Anaconda**.

***

 **1.1. Requisitos previos**

*   Python instalado (3.7 o superior). 
*   pip correctamente configurado. 

***

 **1.2. Instalación con pip (método recomendado por Project Jupyter)**

**Paso 1. Verificar pip**

```sh
pip --version
```

Si pip no está instalado:

```sh
python -m ensurepip --upgrade
```

 **Paso 2. Instalar Jupyter Notebook**

```sh
python -m pip install notebook
```


 **Paso 3. Ejecutar Jupyter**

```sh
jupyter notebook
```

Esto abrirá Jupyter en el navegador predeterminado. 

***

**1.3. Instalación mediante JupyterLab (interfaz moderna)**

```sh
pip install jupyterlab
jupyter lab
```


***

 **1.4. Instalación con Anaconda (método más simple y completo)**

**Paso 1. Abrir Anaconda Navigator**

**Paso 2. Buscar “Jupyter Notebook” y dar clic en Install**


**Paso 3. Lanzar Jupyter Notebook desde Navigator**

***

**2. Instalación de Jupyter en Ubuntu Linux**

Puedes instalar Jupyter en Ubuntu mediante **pip**, **apt** o **Anaconda**. A continuación, se documentan los métodos oficiales.

***

**2.1. Requisitos previos**

Actualizar repositorios e instalar Python:

```sh
sudo apt update
sudo apt install python3 python3-pip python3-venv
```


***

**2.2. Instalación de Jupyter usando pip (recomendado)**

**Paso 1. Crear un entorno virtual**

```sh
python3 -m venv jupyter-env
source jupyter-env/bin/activate
```

 **Paso 2. Instalar Jupyter Notebook**

```sh
pip install notebook
```


 **Paso 3. Ejecutar Jupyter**

```sh
jupyter notebook
```


***

 **2.3. Instalación mediante JupyterLab**

```sh
pip install jupyterlab
jupyter lab
```

***

 **2.4. Instalación usando apt (alternativa simple)**

Disponible en Ubuntu 20.04+:

```sh
sudo apt install jupyter-notebook
```

Ejecutar:

```sh
jupyter notebook
```

***

**2.5. Instalación con Anaconda en Ubuntu**

Descargar Anaconda y ejecutarlo:

```sh
bash Anaconda3-*.sh
```


***

**3. Instalación y uso de Jupyter en Docker**

Docker es uno de los métodos más robustos para ejecutar Jupyter en ambientes aislados y reproducibles.

***

**3.1. Requisitos**

*   Docker instalado. 

***

**3.2. Método 1: Usar la imagen oficial de Jupyter**

**Paso 1. Descargar la imagen**

```sh
docker pull jupyter/base-notebook
```

**Paso 2. Ejecutar el contenedor**

```sh
docker run -it --rm -p 8888:8888 jupyter/base-notebook
```

El contenedor mostrará una URL con token de acceso. Copiarla en el navegador.   

***

**3.3. Método 2: Usar contenedores con montado de volúmenes (para persistencia)**

```sh
docker run -p 8888:8888 -v $(pwd):/home/jovyan/work jupyter/base-notebook
```

Guarda notebooks en tu carpeta local.   

***

**3.4. Método 3: Usar imágenes avanzadas**

Ejemplos:

*   `jupyter/scipy-notebook`
*   `jupyter/datascience-notebook`
*   `jupyter/tensorflow-notebook`    

Ejecutarlas:

```sh
docker run -p 8888:8888 jupyter/scipy-notebook
```

***

**3.5. Método 4: JupyterLab en Docker**

```sh
docker run --rm -p 8889:8888 quay.io/jupyter/base-notebook start-notebook.py --NotebookApp.token='my-token'
```

Acceder vía:

    http://localhost:8889/lab?token=my-token


***

**3.6. Docker Compose**

Ejemplo básico:

```yaml
services:
  jupyter:
    image: jupyter/base-notebook
    ports:
      - 8888:8888
    volumes:
      - jupyter-data:/home/jovyan/work
volumes:
  jupyter-data:
```

Ejecutar:

```sh
docker compose up --build
```

