#### Casos prácticos de análisis y visualización de datos.

> **Casos**
>
> 1.  **Streaming IoT (Kafka + PySpark Structured Streaming)**: agregación con ventanas, *watermarks*, y **detección de anomalías** por ventana.
> 2.  **Clickstream a escala (Spark SQL/DF)**: **sessionización**, **funnel** de conversión, y **matriz de transiciones** (heatmap).

***

Caso 1 — Detección de anomalías en telemetría IoT en **streaming** (Kafka + PySpark Structured Streaming)

Objetivo

Procesar millones de eventos IoT por hora desde Kafka, limpiar/normalizar, agregar por **ventanas deslizantes** y detectar **anomalías** a nivel ventana (e.g., valores que superan *μ + 3σ*). Persistimos métricas en **Delta/Parquet** o **memoria** para visualización.

Arquitectura (conceptual)

**Dispositivos IoT → Kafka (tópico: `iot_telemetry`) → PySpark Structured Streaming → Agregación con ventanas + anomalías → Sink (Delta/Parquet o memoria) → Visualización**

***

Código (comentado y listo para adaptar)

> **Notas**
>
> *   Requiere Spark con conector Kafka. Ajusta la versión de Spark/Kafka según tu entorno.
> *   Para clase, puedes simular Kafka o reemplazar por un *file source* con `readStream.format("json")` y `maxFilesPerTrigger`.

```python
# ============================================================
# CASO 1: Streaming IoT con detección de anomalías por ventana
# ============================================================
from pyspark.sql import SparkSession, functions as F, types as T

# ---------- 1) SparkSession con soporte Kafka ----------
spark = (
    SparkSession.builder
    .appName("Streaming_IoT_Anomalias")
    # Ajusta a la versión de tu clúster; ejemplo con Spark 3.5.x y Scala 2.12:
    .config("spark.jars.packages", "org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.1")
    .getOrCreate()
)
spark.sparkContext.setLogLevel("WARN")

# ---------- 2) Esquema de los mensajes JSON ----------
# Ejemplo de payload:
# {"device_id":"dev-001","ts":"2026-02-15T23:21:00Z","temp":37.2,"vibration":0.03,"cpu":21.5}
telemetry_schema = T.StructType([
    T.StructField("device_id", T.StringType(), True),
    T.StructField("ts", T.StringType(), True),            # ISO 8601
    T.StructField("temp", T.DoubleType(), True),
    T.StructField("vibration", T.DoubleType(), True),
    T.StructField("cpu", T.DoubleType(), True)
])

# ---------- 3) Fuente: Kafka (tópico "iot_telemetry") ----------
# Para demo sin Kafka, reemplaza este bloque con readStream de archivos:
# df_raw = (spark.readStream.format("json")
#   .schema(telemetry_schema)
#   .option("maxFilesPerTrigger", 1)
#   .load("/ruta/a/dataset/json/"))
df_raw = (
    spark.readStream
        .format("kafka")
        .option("kafka.bootstrap.servers", "kafka-broker:9092")
        .option("subscribe", "iot_telemetry")
        .option("startingOffsets", "latest")
        .load()
)

# ---------- 4) Parseo + tipificación + watermark ----------
df = (
    df_raw
    .select(F.col("value").cast("string").alias("json_str"))
    .select(F.from_json("json_str", telemetry_schema).alias("j"))
    .select("j.*")
    .withColumn("event_time", F.to_timestamp("ts"))   # cast a timestamp
    .drop("ts")
    .withWatermark("event_time", "2 minutes")         # tolerancia a lateness
)

# ---------- 5) Agregación por ventanas deslizantes ----------
# Ventana de 1 minuto con deslizamiento de 30 segundos
agg = (
    df.groupBy(
        F.window("event_time", "1 minute", "30 seconds").alias("win"),
        F.col("device_id")
    ).agg(
        F.count("*").alias("n"),
        F.avg("temp").alias("avg_temp"),
        F.stddev_samp("temp").alias("std_temp"),
        F.max("temp").alias("max_temp"),
        F.min("temp").alias("min_temp"),
        F.avg("vibration").alias("avg_vibration"),
        F.avg("cpu").alias("avg_cpu")
    )
    .withColumn("z3_exceeded",
        F.when(F.col("std_temp").isNotNull() &
               (F.col("max_temp") > F.col("avg_temp") + 3*F.col("std_temp")), F.lit(1)).otherwise(F.lit(0))
    )
    .select(
        F.col("device_id"),
        F.col("win.start").alias("window_start"),
        F.col("win.end").alias("window_end"),
        "n","avg_temp","std_temp","max_temp","min_temp","avg_vibration","avg_cpu","z3_exceeded"
    )
)

# ---------- 6) Sink: memoria (para visualización) + checkpoint ----------
# En producción: .format("delta") o "parquet" y .option("path", "/data/iot/agg")
query_memory = (
    agg.writeStream
       .format("memory")                 # tabla temporal en memoria
       .queryName("agg_iot")             # accesible vía spark.sql("select * from agg_iot")
       .outputMode("update")             # solo actualizaciones de ventanas
       .option("checkpointLocation", "/tmp/checkpoints/iot_agg_ckpt")
       .trigger(processingTime="30 seconds")
       .start()
)

# ---------- 7) (Opcional) Visualización periódica desde el sink en memoria ----------
# Para un cuaderno interactivo (Databricks/Notebook): iterar unas veces y mostrar.
import time
for i in range(3):
    time.sleep(35)
    pdf = (
        spark.sql("""
            SELECT device_id, window_start, window_end, n, avg_temp, std_temp, max_temp, min_temp, z3_exceeded
            FROM agg_iot
            ORDER BY window_start DESC, device_id
            LIMIT 200
        """).toPandas()
    )
    if not pdf.empty:
        print(f"Batch {i+1}, filas: {len(pdf)}")
        # Visualización rápida en consola
        print(pdf.head(5))

# query_memory.awaitTermination()  # Descomentar en un entorno real
```

Visualización (gráfica) de anomalías por ventana

> En Big Data, **no** graficamos el *stream completo*; tomamos un **sample** o **últimas ventanas** agregadas.

```python
# Visualización local de las últimas ventanas (muestra)
import matplotlib.pyplot as plt
import seaborn as sns

pdf_last = (
    spark.sql("""
        SELECT device_id, window_start, avg_temp, std_temp, max_temp, z3_exceeded
        FROM agg_iot
        WHERE window_start >= (SELECT max(window_start) FROM agg_iot) - INTERVAL 10 minutes
    """).toPandas()
)

if not pdf_last.empty:
    # Marcamos ventanas con anomalía
    pdf_last["anomaly"] = pdf_last["z3_exceeded"].map({1: "Sí", 0: "No"})

    plt.figure(figsize=(10, 5))
    sns.scatterplot(
        data=pdf_last, x="window_start", y="max_temp",
        hue="anomaly", style="device_id", s=80
    )
    plt.title("IoT: Máxima temperatura por ventana (anomalías marcadas)")
    plt.xlabel("Inicio de ventana")
    plt.ylabel("Temperatura máxima")
    plt.legend()
    plt.tight_layout()
    plt.show()
```

**Puntos técnicos clave**

*   **Watermark** controla el retraso tolerado y limita el estado retenido.
*   **Ventanas deslizantes** permiten *KPIs* temporales finos (1m/30s).
*   Detección **μ ± 3σ** a nivel ventana es simple y eficiente; para *per‑reading* usarías estado avanzado, modelos (e.g., ARIMA/Prophet/IsolationForest) o librerías como **Spark MLlib** o **river** (fuera de Spark).
*   En producción, persistir en **Delta Lake** (transaccional), particionado por `date(window_start)` y `device_id` para *pruning* eficiente.

***

Caso 2 — Clickstream a escala: **sessionización**, **funnel** de conversión y **transiciones** (Spark DataFrames/SQL)

Objetivo

Procesar *logs* web de alto volumen (billones de filas en crudo), crear **sesiones** por usuario, calcular un **embudo** (Home → Product → Cart → Checkout → Purchase) y derivar una **matriz de transiciones** entre páginas para análisis UX.

Esquema típico (Parquet particionado)

`user_id, ts, url, page_type, referrer, device, country, traffic_source, txn_value`

***

Código (comentado y listo para adaptar)

```python
# ============================================================
# CASO 2: Clickstream masivo: sessionización + funnel + transiciones
# ============================================================
from pyspark.sql import SparkSession, functions as F, types as T, Window

spark = (
    SparkSession.builder
    .appName("Clickstream_Sessions_Funnel_Transitions")
    .getOrCreate()
)
spark.sparkContext.setLogLevel("WARN")

# ---------- 1) Carga particionada ----------
# Supón un lakehouse con Parquet particionado por fecha
# Ejemplo de path: "s3://mi-bucket/clickstream/dt=YYYY-MM-DD/"
base_path = "/data/clickstream/"  # ajusta a tu entorno
df = (
    spark.read
         .option("mergeSchema", "true")
         .parquet(base_path)
         .select(
             "user_id",
             F.to_timestamp("ts").alias("ts"),
             "url","page_type","referrer","device","country","traffic_source","txn_value"
         )
         .where("ts >= timestamp '2026-01-01 00:00:00'")
)

# ---------- 2) Sessionización ----------
# Regla: nueva sesión si gap > 30 minutos
w_user_time = Window.partitionBy("user_id").orderBy(F.col("ts").cast("long"))
gap_sec = F.col("ts").cast("long") - F.lag(F.col("ts").cast("long"), 1).over(w_user_time)
is_new_session = F.when((gap_sec.isNull()) | (gap_sec > 30*60), 1).otherwise(0)

df_sess = (
    df.withColumn("new_session", is_new_session)
      .withColumn("session_id",
                  F.sum("new_session").over(w_user_time))   # sesión incremental por user
)

# ---------- 3) Funnel de conversión ----------
# Definimos etapas por page_type
# home -> product -> cart -> checkout -> purchase
from pyspark.sql.functions import expr

df_stage = df_sess.select(
    "user_id","session_id","ts","page_type",
    F.when(F.col("page_type")=="home", 1).otherwise(0).alias("stage_home"),
    F.when(F.col("page_type")=="product", 1).otherwise(0).alias("stage_product"),
    F.when(F.col("page_type")=="cart", 1).otherwise(0).alias("stage_cart"),
    F.when(F.col("page_type")=="checkout", 1).otherwise(0).alias("stage_checkout"),
    F.when(F.col("page_type")=="purchase", 1).otherwise(0).alias("stage_purchase")
)

# Reducimos a nivel sesión-usuario (si visitó al menos una vez cada etapa)
sess_funnel = (
    df_stage.groupBy("user_id","session_id").agg(
        F.max("stage_home").alias("home"),
        F.max("stage_product").alias("product"),
        F.max("stage_cart").alias("cart"),
        F.max("stage_checkout").alias("checkout"),
        F.max("stage_purchase").alias("purchase")
    )
)

# Métricas del funnel (conteo de sesiones que alcanzan la etapa)
funnel_counts = sess_funnel.agg(
    F.sum("home").alias("n_home"),
    F.sum("product").alias("n_product"),
    F.sum("cart").alias("n_cart"),
    F.sum("checkout").alias("n_checkout"),
    F.sum("purchase").alias("n_purchase")
).collect()[0].asDict()

# Tasas de conversión step-by-step
def safe_rate(num, den):
    return float(num)/float(den) if den and den > 0 else 0.0

n_home     = funnel_counts["n_home"]
n_product  = funnel_counts["n_product"]
n_cart     = funnel_counts["n_cart"]
n_checkout = funnel_counts["n_checkout"]
n_purchase = funnel_counts["n_purchase"]

rates = {
    "home→product":  safe_rate(n_product, n_home),
    "product→cart":  safe_rate(n_cart, n_product),
    "cart→checkout": safe_rate(n_checkout, n_cart),
    "checkout→purchase": safe_rate(n_purchase, n_checkout),
    "home→purchase (global)": safe_rate(n_purchase, n_home)
}

print("Funnel (conteos):", funnel_counts)
print("Funnel (tasas):", rates)

# ---------- 4) Matriz de transiciones ----------
# Para cada user-session, par (page_t, next_page_t) con LEAD
w_session = Window.partitionBy("user_id","session_id").orderBy("ts")
df_transitions = (
    df_sess
    .withColumn("next_page", F.lead("page_type").over(w_session))
    .where(F.col("next_page").isNotNull())
    .groupBy("page_type","next_page")
    .count()
    .withColumnRenamed("page_type","from_page")
    .withColumnRenamed("next_page","to_page")
    .orderBy(F.desc("count"))
)

# Limitamos a top-N transiciones para visualización
topN = 50
df_top_trans = df_transitions.limit(topN)

# ---------- 5) Export a Pandas para visualización ----------
pdf_funnel = spark.createDataFrame([
    ("home", n_home),
    ("product", n_product),
    ("cart", n_cart),
    ("checkout", n_checkout),
    ("purchase", n_purchase),
], ["stage","count"]).toPandas()

pdf_trans = df_top_trans.toPandas()
```

Visualizaciones (funnel y matriz de transiciones)

```python
import matplotlib.pyplot as plt
import seaborn as sns
import pandas as pd

# ------ Funnel: barras ------
plt.figure(figsize=(7,4))
sns.barplot(data=pdf_funnel, x="stage", y="count", palette="Blues_d")
plt.title("Funnel de conversión por etapa (conteos de sesiones)")
plt.xlabel("Etapa")
plt.ylabel("Número de sesiones")
plt.tight_layout()
plt.show()

# ------ Matriz de transiciones (heatmap) ------
# Construimos una matriz from_page x to_page con conteos
if not pdf_trans.empty:
    pivot = pdf_trans.pivot_table(index="from_page", columns="to_page", values="count", aggfunc="sum", fill_value=0)
    plt.figure(figsize=(10,7))
    sns.heatmap(pivot, annot=False, cmap="Reds", cbar_kws={"label": "Conteo transiciones"})
    plt.title("Matriz de transiciones (top N)")
    plt.xlabel("To page")
    plt.ylabel("From page")
    plt.tight_layout()
    plt.show()
```

**Puntos técnicos clave**

*   **Sessionización** con `lag` y acumulador (`sum(new_session)`) es un patrón estándar para *gap‑based sessions*.
*   El **funnel** se evalúa a nivel **sesión** (no solo usuario) para medir intención en ventanas temporales coherentes.
*   Para transiciones, usamos `lead` dentro de `(user_id, session_id)` ordenado por `ts` → *pares ordenados* (from→to).
*   Escalabilidad:
    *   **Particiona** por `dt`/`country`/`traffic_source`.
    *   Usa **Parquet/Delta** con **file sizes** de 128–512 MB.
    *   **Bloom/Z‑Order** (si usas Delta) sobre `user_id` o `dt`.
    *   `cache/persist` solo etapas reuse-intensive.
    *   *Broadcast joins* solo cuando sea seguro (dim << 10^7 filas).

***

Cómo integrarlo en tu materia (sugerencia didáctica)

*   **Actividad guiada**:
    *   Proveer un **dataset sintético** (IoT y clickstream) en Parquet.
    *   Estudiantes ejecutan los *pipelines*, cambian ventana (1m→2m), evalúan impacto en **recall** de anomalías y estabilidad de KPIs.
    *   En clickstream, modificar `gap` (30→20 minutos) y comparar **número de sesiones** y **tasa de conversión**.

*   **Extensiones**:
    1.  Sustituir *μ±3σ* por **MAD** (Median Absolute Deviation) o **EWMA**.
    2.  Añadir **Sankey** (Plotly) para el funnel, o **graph** de transiciones (NetworkX).
    3.  Persistir en **Delta Lake** con **time travel** para reproducibilidad del laboratorio.

*   **Evaluación**:
    *   Reporte con **métricas** (tasa de anomalías por device, funnel por país/segmento), **gráficas** y **decisiones** (ej. rediseño de navegación, umbral adaptativo por device).

***

Requisitos y ejecución

*   **Infra**: Spark 3.3+ (ideal 3.5.x), Python 3.9+, almacenamiento (S3/ADLS/HDFS), opcional **Kafka**.
*   **Paquetes**: `pyspark`, `matplotlib`, `seaborn`. (Opcional: `delta-spark`, `plotly`, `networkx`).
