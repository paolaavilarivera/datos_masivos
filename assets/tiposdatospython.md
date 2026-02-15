#### **Tabla: Tipos de datos que se pueden recopilar con Python**

| **Tipo de dato**                     | **Descripción**                                                                | **Ejemplos de uso / fuentes**                                                    |
| ------------------------------------ | ------------------------------------------------------------------------------ | -------------------------------------------------------------------------------- |
| **Numéricos**                        | Datos enteros, flotantes o números con precisión arbitraria.                   | Medición de sensores, conteos, estadísticas, valores financieros, lecturas IoT.  |
| **Categóricos**                      | Etiquetas o valores discretos que representan categorías.                      | País, género, tipo de producto, estado de un pedido.                             |
| **Textuales**                        | Información almacenada como cadenas de caracteres.                             | Comentarios de usuarios, artículos, tweets, logs de sistemas, correos.           |
| **Booleanos**                        | Valores lógicos de verdadero/falso.                                            | Flags de validación, indicadores de estado, activación/desactivación de eventos. |
| **Fechas y tiempos (DateTime)**      | Momentos específicos o intervalos temporales.                                  | Tiempos de transacciones, eventos de sensores, marcas de tiempo en logs.         |
| **Datos estructurados (tabulares)**  | Datos organizados en filas y columnas con formato definido.                    | CSV, Excel, tablas SQL, reportes de negocio.                                     |
| **Datos semiestructurados**          | Información con estructura flexible, basada en pares clave–valor o jerarquías. | JSON, XML, YAML, APIs REST, documentos NoSQL.                                    |
| **Datos no estructurados**           | Datos sin un formato fijo o fácilmente tableable.                              | Texto libre, imágenes, audio, video, PDFs, HTML.                                 |
| **Series temporales**                | Datos ordenados cronológicamente donde el tiempo es un eje principal.          | Precios bursátiles, tráfico de red, clima, telemetría IoT.                       |
| **Datos geoespaciales**              | Información con coordenadas o forma espacial.                                  | Coordenadas GPS, mapas, polígonos GIS, rutas de transporte.                      |
| **Datos de sensores / IoT**          | Lecturas provenientes de dispositivos físicos y sensores.                      | Temperatura, humedad, presión, acelerómetros, RFID.                              |
| **Datos transaccionales**            | Registros de operaciones o actividades.                                        | Ventas, pagos, clics, movimientos bancarios.                                     |
| **Datos multimedia**                 | Imágenes, audio, video o streams digitales.                                    | Reconocimiento de imágenes, análisis de voz, videovigilancia.                    |
| **Grafos y redes**                   | Datos que representan relaciones entre entidades.                              | Redes sociales, grafos de conocimiento, rutas de red.                            |
| **Datos en tiempo real (streaming)** | Datos recibidos continuamente y procesados al vuelo.                           | Kafka, MQTT, websockets, telemetría industrial.                                  |

***

> **Nota**: algunos ejemplos requieren instalar paquetes externos (indicados). Realiza la instalación en tu entorno (p. ej., `pip install paquete`) antes de ejecutarlos.

***

Datos **numéricos**

```python
# Lectura de numéricos desde CSV y cálculo básico
import pandas as pd
import numpy as np

df = pd.read_csv("datos/sensores.csv")  # columnas: timestamp, temperatura, presion
temperaturas = df["temperatura"].to_numpy(dtype=np.float64)
print("Promedio de temperatura:", temperaturas.mean())
```

***

Datos **categóricos**

```python
import pandas as pd

df = pd.read_csv("datos/clientes.csv")  # columnas: id, pais, genero, segmento
categorias = df["segmento"].astype("category")
print(categorias.value_counts())
```

***

Datos **textuales**

```python
# Lectura de texto libre desde archivos y limpieza básica
from pathlib import Path

texto = Path("datos/comentarios.txt").read_text(encoding="utf-8")
# Normalización trivial
texto = texto.lower().replace("\n", " ").strip()
print(texto[:200], "...")
```

***

Datos **booleanos**

```python
import pandas as pd

df = pd.read_json("datos/flags.json")  # {"activo": true, "enviado": false, ...}
df["activo"] = df["activo"].astype(bool)
print(df.dtypes)
```

***

**Fechas y tiempos** (DateTime)

```python
import pandas as pd

df = pd.read_csv("datos/transacciones.csv", parse_dates=["fecha"])
# Filtrar periodo
rango = df[(df["fecha"] >= "2026-01-01") & (df["fecha"] < "2026-02-01")]
print(rango.head())
```

***

Datos **estructurados (tabulares)**

```python
# Lectura tabular desde Excel y SQL
import pandas as pd
import sqlite3

# Excel
ventas = pd.read_excel("datos/reporte.xlsx", sheet_name="Ene", engine="openpyxl")

# SQL (SQLite como ejemplo)
con = sqlite3.connect("datos/tienda.db")
top = pd.read_sql_query("""
    SELECT producto, SUM(unidades) AS total
    FROM ventas
    GROUP BY producto
    ORDER BY total DESC
    LIMIT 5
""", con)
con.close()

print(top)
```

***

Datos **semiestructurados** (JSON, XML, YAML)

```python
# JSON local o de API
import json, requests

# Local
with open("datos/config.json", "r", encoding="utf-8") as f:
    cfg = json.load(f)

# API (JSON)
r = requests.get("https://api.example.com/v1/items", timeout=10)
r.raise_for_status()
items = r.json()
print("Items recibidos:", len(items))
```

```python
# XML: leer y extraer campos
import xml.etree.ElementTree as ET

tree = ET.parse("datos/catalogo.xml")
root = tree.getroot()
titulos = [n.text for n in root.findall(".//libro/titulo")]
print(titulos[:3])
```

```python
# YAML
# pip install pyyaml
import yaml

cfg = yaml.safe_load(open("datos/pipeline.yaml", "r", encoding="utf-8"))
print(cfg["jobs"][0]["name"])
```

***

Datos **no estructurados** (PDF, HTML)

```python
# PDF: extraer texto (básico)
# pip install pypdf2
from PyPDF2 import PdfReader

reader = PdfReader("docs/informe.pdf")
texto = []
for page in reader.pages:
    texto.append(page.extract_text() or "")
print("Primeros 300 chars:", "".join(texto)[:300])
```

```python
# HTML: scraping simple
# pip install beautifulsoup4 lxml requests
import requests
from bs4 import BeautifulSoup

resp = requests.get("https://example.com/noticias", timeout=10)
soup = BeautifulSoup(resp.text, "lxml")
titulares = [h.get_text(strip=True) for h in soup.select("h2.titular")]
print(titulares[:5])
```

***

**Series temporales**

```python
import pandas as pd

ts = pd.read_csv("datos/precios.csv", parse_dates=["fecha"], index_col="fecha").sort_index()
# Resampleo: promedio diario
diario = ts["precio"].resample("D").mean()
print(diario.head())
```

***

Datos **geoespaciales**

```python
# pip install geopandas shapely pyproj fiona
import geopandas as gpd

# GeoJSON o Shapefile
gdf = gpd.read_file("datos/colonias.geojson")  # columnas: geometry, nombre, municipio
# Filtro espacial (ejemplo: bounding box)
bb = gpd.GeoDataFrame(geometry=gpd.GeoSeries.from_bbox((-102.5, 21.7, -102.2, 21.95)), crs="EPSG:4326")
en_bbox = gdf[gdf.within(bb.iloc[0].geometry)]
print(en_bbox[["nombre", "geometry"]].head())
```

***

Datos de **sensores / IoT**

```python
# MQTT (IoT)
# pip install paho-mqtt
import json
import paho.mqtt.client as mqtt

def on_message(client, userdata, msg):
    data = json.loads(msg.payload.decode("utf-8"))
    print("Sensor:", data.get("id"), "Temp:", data.get("temp"))

cli = mqtt.Client(client_id="colector-iot")
cli.on_message = on_message
cli.connect("broker.local", 1883, 60)
cli.subscribe("planta/sensores/temperatura")
cli.loop_start()
```

***

Datos **transaccionales**

```python
# Consulta a DB transaccional (ej. PostgreSQL)
# pip install sqlalchemy psycopg2-binary
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine("postgresql+psycopg2://user:pass@host:5432/erp")
q = """
SELECT order_id, customer_id, amount, created_at
FROM orders
WHERE created_at::date = CURRENT_DATE
"""
transacciones_hoy = pd.read_sql(q, engine)
print(transacciones_hoy.head())
```

***

Datos **multimedia** (imágenes, audio, video)

```python
# Imagen: cargar y convertir a numpy
# pip install pillow numpy
from PIL import Image
import numpy as np

img = Image.open("imagenes/foto.jpg").convert("RGB")
arr = np.asarray(img)  # (alto, ancho, 3)
print(arr.shape, arr.dtype)
```

```python
# Audio: lectura de waveform y sample rate
# pip install soundfile
import soundfile as sf

data, sr = sf.read("audio/grabacion.wav")
print("Muestras:", len(data), "Frecuencia (Hz):", sr)
```

***

**Grafos y redes**

```python
# pip install networkx
import networkx as nx

G = nx.read_edgelist("datos/red_edges.csv", delimiter=",", nodetype=str, data=(("peso", float),))
print("Nodos:", G.number_of_nodes(), "Aristas:", G.number_of_edges())
# Recopilar métricas básicas
deg = dict(G.degree())
print("Grado medio:", sum(deg.values()) / len(deg))
```

***

Datos en **tiempo real (streaming)**

```python
# Kafka: consumidor básico
# pip install kafka-python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    "events_topic",
    bootstrap_servers=["kafka1:9092"],
    auto_offset_reset="latest",
    value_deserializer=lambda v: json.loads(v.decode("utf-8")),
    group_id="grupo-analitica"
)

for msg in consumer:
    evento = msg.value
    print("Evento:", evento.get("type"), "TS:", evento.get("ts"))
```

***

Lectura desde **nube/objetos** (S3/GCS/Azure)

```python
# S3 con pandas + s3fs
# pip install pandas s3fs
import pandas as pd

df = pd.read_parquet("s3://mi-bucket/dwh/ventas.parquet", storage_options={"anon": False})
print(df.head())
```

***

Formatos columnares **Parquet / Feather / HDF5**

```python
# Parquet
# pip install pyarrow pandas
import pandas as pd

df = pd.read_parquet("datos/ventas.parquet")  # rápido y columnar
# Feather
df2 = pd.read_feather("datos/ventas.feather")
print(len(df), len(df2))
```

```python
# HDF5
# pip install tables
store = pd.HDFStore("datos/series.h5", mode="r")
print(store.keys())
serie = store["/serie1"]
store.close()
```

***

Consejos de **buenas prácticas**

*   Usa **entornos virtuales** (`venv`/`conda`) y **pinning** de versiones.
*   Centraliza **credenciales** en variables de entorno o *secret managers*.
*   Para lotes grandes: formatos **columnar** (Parquet/Feather) y **particionamiento** (por fecha/cliente).
*   Para streaming: maneja **backpressure**, **reintentos** y **idempotencia**.

***

