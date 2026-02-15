#### **Tabla: Fuentes de datos comunes en Python**

| **Fuente de Datos**                                                | **Descripción**                                                                                            | **Uso típico en análisis de datos**                                                                          |
| ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **Archivos CSV**                                                   | Datos tabulares separados por comas. Es el formato más usado en análisis por su simplicidad.               | Lectura y análisis de tablas, series temporales, reportes exportados, datos limpios o semiestructurados.     |
| **Archivos Excel (XLS/XLSX)**                                      | Hojas de cálculo con múltiples pestañas, formatos y tipos de datos.                                        | Integración con bases administrativas, reportes empresariales, seguimiento de inventarios o ventas.          |
| **Bases de datos SQL (MySQL, PostgreSQL, SQLite, entre otros)**           | Sistemas relacionales donde los datos se organizan en tablas con relaciones definidas.                     | Consultas estructuradas, análisis de grandes volúmenes transaccionales, integración con sistemas de negocio. |
| **Bases NoSQL (MongoDB, Cassandra, Redis, entre otros)**                  | Bases orientadas a documentos, clave–valor o grafos. Diseñadas para datos no estructurados o distribuidos. | Análisis de logs, datos semiestructurados (JSON), telemetría, grandes volúmenes distribuidos.                |
| **APIs REST/JSON**                                                 | Servicios web que permiten acceder a datos mediante solicitudes HTTP.                                      | Consumo de datos dinámicos como clima, finanzas, redes sociales, dashboards en tiempo real.                  |
| **Archivos JSON**                                                  | Formato jerárquico para intercambiar datos semiestructurados.                                              | Datos provenientes de APIs, configuraciones, datos anidados o colecciones complejas.                         |
| **Archivos de texto (TXT, LOG)**                                   | Datos sin estructura o con separación mediante delimitadores.                                              | Análisis de registros, limpieza de datos crudos, análisis de logs de sistemas.                               |
| **HDF5 / Parquet / Feather**                                       | Formatos optimizados para análisis de datos a gran escala.                                                 | Big Data, procesamiento distribuido, almacenamiento columnar eficiente para alto rendimiento.                |
| **Datasets en la nube (AWS S3, Google Cloud Storage, Azure Blob)** | Almacenamiento remoto y escalable con acceso programático.                                                 | Pipelines de datos, integración con sistemas distribuidos, análisis masivo.                                  |
| **Web Scraping (HTML)**                                            | Datos extraídos directamente de páginas web mediante técnicas de raspado.                                  | Recolección de información pública, precios, noticias, catálogos y contenido no disponible vía API.          |
| **Flujos de datos en tiempo real (Kafka, MQTT, sockets)**          | Fuentes continuas y rápidas de datos.                                                                      | Monitoreo, IoT, analítica en streaming, detección de eventos.                                                |

***


> **Notas**
>
> *   Para mantener los ejemplos simples, todos leen o extraen datos y muestran un *snippet* de uso.
> *   Indico los **paquetes necesarios** y alternativas cuando aplica.
> *   No ejecutes `pip install` dentro del código de tu app; úsalo en tu entorno/terminal (o en un `requirements.txt`).

***

1) Archivos CSV

**Paquetes**: `pandas`

```python
import pandas as pd

# Lectura
df = pd.read_csv("datos/ventas.csv")  # admite sep, encoding, dtype, parse_dates...
print(df.head())

# Escritura
df_filtrado = df[df["unidades"] > 100]
df_filtrado.to_csv("out/ventas_filtradas.csv", index=False)
```

***

 2) Archivos Excel (XLS/XLSX)

**Paquetes**: `pandas`, motores: `openpyxl` (xlsx), `xlrd` (xls - según versión)

```python
import pandas as pd

# Leer hoja específica y columnas
df = pd.read_excel("datos/reporte.xlsx", sheet_name="Ene", engine="openpyxl", usecols="A:D")
print(df.describe())

# Escribir a varias hojas
with pd.ExcelWriter("out/reportes.xlsx", engine="openpyxl") as xw:
    df.to_excel(xw, sheet_name="Resumen", index=False)
```

***

 3) Bases de datos SQL (SQLite / PostgreSQL / MySQL)

**Paquetes**: `sqlite3` (stdlib) o `SQLAlchemy` + driver (`psycopg2-binary`, `mysqlclient`, etc.)

```python
import sqlite3
import pandas as pd

# SQLite local
con = sqlite3.connect("datos/tienda.db")
df = pd.read_sql_query("""
    SELECT producto, SUM(unidades) AS total
    FROM ventas
    WHERE fecha >= '2026-01-01'
    GROUP BY producto
    ORDER BY total DESC
    LIMIT 10
""", con)
con.close()

print(df)
```

> **PostgreSQL con SQLAlchemy**  
> `pip install sqlalchemy psycopg2-binary`  
> `engine = create_engine("postgresql+psycopg2://user:pass@host:5432/db")` y `pd.read_sql(query, engine)`.

***

 4) Bases NoSQL (MongoDB)

**Paquetes**: `pymongo`

```python
from pymongo import MongoClient

client = MongoClient("mongodb://usuario:password@host:27017/?authSource=admin")
db = client["analytics"]
col = db["eventos"]

# Consulta por rango de fechas y campo anidado
cursor = col.find(
    {"timestamp": {"$gte": "2026-01-01"}, "meta.canal": "web"},
    {"_id": 0, "usuario": 1, "evento": 1, "timestamp": 1}
).limit(5)

for doc in cursor:
    print(doc)
```

***

 5) APIs REST/JSON

**Paquetes**: `requests`

```python
import requests

url = "https://api.example.com/v1/precios"
params = {"sku": "ABC123", "moneda": "MXN"}
headers = {"Authorization": "Bearer <TOKEN>"}

r = requests.get(url, params=params, headers=headers, timeout=10)
r.raise_for_status()
data = r.json()  # dict/list en memoria

print(data.get("precio_actual"), data.get("fecha"))
```

***

 6) Archivos JSON (locales)

**Paquetes**: `json` (stdlib), o `pandas` si tabla

```python
import json

with open("datos/config.json", "r", encoding="utf-8") as f:
    cfg = json.load(f)

print(cfg["pipeline"]["batch_size"])

# Si es una colección homogénea tabular:
import pandas as pd
df = pd.read_json("datos/ventas.json", lines=False)  # o lines=True para JSON Lines
print(df.head())
```

***

 7) Archivos de texto / LOG

**Paquetes**: stdlib (`pathlib`, `re`), o `pandas` si delimitado

```python
from pathlib import Path
import re

pat = re.compile(r"\[(?P<ts>.+?)\]\s+(?P<level>INFO|WARN|ERROR)\s+(?P<msg>.+)")
rows = []
for line in Path("logs/app.log").read_text(encoding="utf-8").splitlines():
    m = pat.search(line)
    if m:
        rows.append(m.groupdict())

# A DataFrame si deseas analizar
import pandas as pd
df = pd.DataFrame(rows)
print(df["level"].value_counts())
```

***

 8) Formatos columnares / científicos (HDF5, Parquet, Feather)

**Paquetes**: `pandas`, `pyarrow` (Parquet/Feather), `tables` o `h5py` (HDF5)

```python
import pandas as pd

# Parquet (columna, compresión, ideal para big data local)
df = pd.read_parquet("datos/ventas.parquet")  # requiere pyarrow
print(df.memory_usage(deep=True).sum())

# Convertir CSV -> Parquet
df_csv = pd.read_csv("datos/ventas.csv")
df_csv.to_parquet("out/ventas.parquet", index=False, compression="snappy")
```

***

 9) Datasets en la nube (AWS S3, GCS, Azure Blob)

**Paquetes**:

*   **S3**: `pandas`, `s3fs` (para rutas `s3://`), o `boto3` para control detallado.
*   **GCS**: `gcsfs`
*   **Azure**: `adlfs`

```python
import pandas as pd

# Leer directamente desde S3 (recomendado: credenciales por perfil/variables)
df = pd.read_csv("s3://mi-bucket/datos/ventas.csv", storage_options={
    "anon": False,  # usar credenciales configuradas en el entorno
})
print(df.head())
```

> **Con boto3 (control fino)**

```python
import boto3, io
import pandas as pd

s3 = boto3.client("s3")
obj = s3.get_object(Bucket="mi-bucket", Key="datos/ventas.csv")
df = pd.read_csv(io.BytesIO(obj["Body"].read()))
```

***

 10) Web Scraping (HTML)

**Paquetes**: `requests`, `beautifulsoup4` (BS4), opcional `lxml`

```python
import requests
from bs4 import BeautifulSoup

resp = requests.get("https://example.com/catalogo", timeout=10)
resp.raise_for_status()

soup = BeautifulSoup(resp.text, "lxml")
items = []
for card in soup.select(".producto-card"):
    items.append({
        "titulo": card.select_one(".titulo").get_text(strip=True),
        "precio": card.select_one(".precio").get_text(strip=True),
        "link":   card.select_one("a")["href"]
    })

print(items[:3])
```

> **Consejos**: respeta robots.txt, rate limiting, y términos de uso; usa cabeceras y *backoff*.

***

 11) Flujos de datos en tiempo real (Kafka / MQTT / sockets)

a) **Kafka**

**Paquetes**: `kafka-python` o `confluent-kafka`

```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    "topic_eventos",
    bootstrap_servers=["kafka1:9092"],
    value_serializer=None,
    value_deserializer=lambda v: json.loads(v.decode("utf-8")),
    auto_offset_reset="earliest",
    enable_auto_commit=True,
    group_id="analitica-grupo1",
)

for msg in consumer:
    evento = msg.value
    # Procesa/filtra/enriquece
    print(evento)
```

 b) **MQTT (IoT)**

**Paquetes**: `paho-mqtt`

```python
import json
import paho.mqtt.client as mqtt

def on_message(client, userdata, msg):
    payload = json.loads(msg.payload.decode("utf-8"))
    print("Temperatura:", payload["temp"])

cli = mqtt.Client(client_id="analizador-1")
cli.on_message = on_message
cli.connect("broker.iot.local", 1883, 60)
cli.subscribe("sala/temperatura")
cli.loop_forever()
```

***

Extras útiles

*   **Validación de esquemas**: `pydantic`, `pandera`.
*   **Conexiones seguras**: variables de entorno (`os.getenv`) y *secrets managers*.
*   **Batch vs. streaming**: usa *parquet* y *particionamiento* en batch; *buffers* y *backpressure* en streaming.
*   **Reproducibilidad**: `requirements.txt`/`poetry.lock`, `venv/conda`, y *pinning* de versiones.


