#### Interpretación de datos inferenciales y modelación estadística.

1.  **Inferencia** (prueba t de Welch a partir de agregados distribuidos —sin traer todo a memoria—).
2.  **Modelación estadística** a gran escala:
    *   **Regresión lineal** y **logística** con aprendizaje incremental (*online*) usando `SGDRegressor` / `SGDClassifier`.
    *   **Intervalos de confianza (IC)** de los coeficientes mediante **bootstrap distribuido**.
3.  Dos rutas de ejecución:
    *   **Opción A (sin clúster)**: **Dask DataFrame** + scikit‑learn (*online*).
    *   **Opción B (con clúster)**: **PySpark** (lectura, EDA agregada, LR/Logit con MLlib).

> **Por qué así**: en Big Data no conviene muestrear todo a memoria ni ajustar modelos que requieran matrices densas completas. En su lugar:
>
> *   Para **inferencias clásicas** (p. ej., t de Welch), basta con **agregados por grupo** (n, media, varianza).
> *   Para **modelos** a escala, usamos **aprendizaje incremental** (SGD) o frameworks distribuidos (Spark MLlib).

***

Requisitos (sugeridos)

*   Opción A: `pip install dask[dataframe] pandas numpy scipy scikit-learn`
*   Opción B: `pip install pyspark` (o ejecutar en un clúster Spark ya provisionado)

***

Opción A — Dask + scikit‑learn (*online*)

Escenario

Tenemos un *dataset* parquet particionado (p. ej., `s3://mi-bucket/ventas/fecha=YYYY-MM-DD/*.parquet`) con columnas:

*   `tratamiento` (0/1), `ingreso` (continuo), `precio`, `publicidad`, `dia`, `sucursal` (categóricas).

Código (end‑to‑end, con comentarios)

```python
# ============================================
# 0) Imports y helpers
# ============================================
import numpy as np
import pandas as pd
import dask.dataframe as dd
from dask.diagnostics import ProgressBar

from scipy import stats
from sklearn.linear_model import SGDRegressor, SGDClassifier
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import r2_score, roc_auc_score
from sklearn.utils import resample

import warnings
warnings.filterwarnings("ignore")

ProgressBar().register()  # barra de progreso para .compute()

# ============================================
# 1) Carga distribuida (Parquet / S3 / local)
#    - Cambia 'path' por tu ruta real (local o s3://)
# ============================================
path = "data/ventas/*.parquet"   # ejemplo local; para S3 usa s3://bucket/...
storage_options = None           # {"anon": False} si usas S3 con credenciales del entorno

ddf = dd.read_parquet(path, storage_options=storage_options)

# Selecciona columnas necesarias y tipifica
cols = ["tratamiento", "ingreso", "precio", "publicidad", "dia", "sucursal"]
ddf = ddf[cols].astype({
    "tratamiento": "int8",
    "ingreso": "float64",
    "precio": "float64",
    "publicidad": "float64",
    "dia": "category",
    "sucursal": "category",
})

# Limpieza mínima
# (Ejemplo: descartar negativos imposibles o NaN críticos sin traer todo a memoria)
ddf = ddf[(ddf["ingreso"] > 0) & (ddf["precio"] > 0) & (ddf["publicidad"] >= 0)]
ddf = ddf.dropna(subset=["ingreso","precio","publicidad","tratamiento"])

# ============================================
# 2) Inferencia: t de Welch a partir de agregados
#    - No traemos filas; sólo agregados por grupo
# ============================================
aggs = ddf.groupby("tratamiento").agg(
    n=("ingreso","count"),
    mean=("ingreso","mean"),
    var=("ingreso","var")
).compute()  # trae un DataFrame pequeño (2 filas) al driver

# Extrae estadísticos por grupo
n0, m0, v0 = int(aggs.loc[0, "n"]), aggs.loc[0, "mean"], aggs.loc[0, "var"]
n1, m1, v1 = int(aggs.loc[1, "n"]), aggs.loc[1, "mean"], aggs.loc[1, "var"]

# Estadístico t de Welch (sólo con agregados)
t_num = (m1 - m0)
t_den = np.sqrt(v1/n1 + v0/n0)
t_stat = t_num / t_den

# Grados de libertad de Welch–Satterthwaite
df_num = (v1/n1 + v0/n0)**2
df_den = (v1**2 / (n1**2*(n1-1))) + (v0**2 / (n0**2*(n0-1)))
df_welch = df_num / df_den

# p-valor (bilateral) e IC95% de la diferencia de medias
p_val = 2*stats.t.sf(np.abs(t_stat), df=df_welch)
diff = m1 - m0
# error estándar de la diferencia
se_diff = t_den
# IC95%
alpha = 0.05
t_crit = stats.t.ppf(1 - alpha/2, df=df_welch)
ic_low = diff - t_crit*se_diff
ic_high = diff + t_crit*se_diff

print(f"\n[Inferencia] t (Welch) = {t_stat:.3f}, gl={df_welch:.1f}, p={p_val:.3g}")
print(f"Diferencia de medias (trat - ctrl) = {diff:.2f}  IC95% [{ic_low:.2f}, {ic_high:.2f}]")

# Tamaño de efecto (Cohen's d) usando varianza agrupada aproximada
sp = np.sqrt(((v1*(n1-1) + v0*(n0-1)) / (n1 + n0 - 2)))
cohen_d = diff / sp
print(f"Cohen's d ≈ {cohen_d:.3f}")

# ============================================
# 3) Ingeniería de variables a escala
#    - One-hot mínimo y ensamblado de features
#    - Sin .compute(): se crea un grafo perezoso
# ============================================
# Dummy-encoding controlado (sólo para variables con cardinalidad moderada)
ddf = dd.get_dummies(ddf, columns=["dia","sucursal"], drop_first=True)

# Feature matrix y target para regresión y clasificación
#   - Regresión: y_reg = ingreso
#   - Clasificación: y_bin = 1 si ingreso >= p75
# Para el umbral P75 calculamos con aproximación cuantil en Dask
p75 = ddf["ingreso"].quantile(0.75).compute()
ddf["y_bin"] = (ddf["ingreso"] >= p75).astype("int8")

feature_cols = [c for c in ddf.columns if c not in ["ingreso","y_bin"]]
print("\nN° features:", len(feature_cols))

# ============================================
# 4) Modelos 'online' (SGD) con mini-lotes por partición
#    - Ajuste incremental recorriendo particiones de Dask
# ============================================
# Estandarización online: ajustamos con una pasada y transformamos "en vuelo".
# (Para un pipeline formal, puedes usar partial_fit de StandardScaler incremental de sklearn>=1.4)
scaler_means = None
scaler_stds = None

# Inicializa modelos
reg = SGDRegressor(loss="squared_error", penalty="l2", alpha=1e-4, learning_rate="invscaling", random_state=42)
clf = SGDClassifier(loss="log_loss", penalty="l2", alpha=1e-4, learning_rate="optimal", random_state=42)

# Obtén lista de particiones retrasadas
parts = ddf[feature_cols + ["ingreso","y_bin"]].to_delayed()

# Pasada 1: estima medias/desv (para estandarizar)
sum_x = None
sum_x2 = None
count = 0
for d in parts:
    part = d.compute()
    Xp = part[feature_cols].astype("float64").values
    if sum_x is None:
        sum_x = Xp.sum(axis=0)
        sum_x2 = (Xp**2).sum(axis=0)
    else:
        sum_x += Xp.sum(axis=0)
        sum_x2 += (Xp**2).sum(axis=0)
    count += len(part)

scaler_means = sum_x / count
scaler_stds = np.sqrt((sum_x2 / count) - scaler_means**2)
scaler_stds[scaler_stds == 0] = 1.0  # evita división por cero

# Pasada 2: entrenamiento incremental
classes_ = np.array([0,1], dtype=np.int64)
for epoch in range(2):  # 2 épocas sobre todas las particiones
    for d in parts:
        part = d.compute()
        Xp = part[feature_cols].astype("float64").values
        # Estandariza on-the-fly
        Xp = (Xp - scaler_means) / scaler_stds

        # --- Regresión (ingreso) ---
        y_reg = part["ingreso"].values
        reg.partial_fit(Xp, y_reg)

        # --- Clasificación (ticket alto) ---
        y_bin = part["y_bin"].values
        clf.partial_fit(Xp, y_bin, classes=classes_)

# ============================================
# 5) Evaluación rápida (usamos una partición como validación hold-out)
# ============================================
val = parts[-1].compute()
Xv = val[feature_cols].astype("float64").values
Xv = (Xv - scaler_means) / scaler_stds
y_reg_v = val["ingreso"].values
y_bin_v = val["y_bin"].values

y_reg_hat = reg.predict(Xv)
y_bin_proba = clf.predict_proba(Xv)[:,1]

print("\n[Regresión] R^2 hold-out:", round(r2_score(y_reg_v, y_reg_hat), 4))
print("[Clasificación] AUC hold-out:", round(roc_auc_score(y_bin_v, y_bin_proba), 4))

# ============================================
# 6) IC de coeficientes por bootstrap distribuido (aprox.)
#    - Repite train en remuestreos de particiones (sin mover todos los datos)
# ============================================
def fit_sgd_on_parts(selected_indices, seed=0):
    r = SGDClassifier(loss="log_loss", penalty="l2", alpha=1e-4, random_state=seed)
    # re-usa scaler estimado globalmente
    for idx in selected_indices:
        part = parts[idx].compute()
        Xp = part[feature_cols].astype("float64").values
        Xp = (Xp - scaler_means) / scaler_stds
        yp = part["y_bin"].values
        r.partial_fit(Xp, yp, classes=classes_)
    return r.coef_.ravel()

# Bootstrap: toma muestras con reemplazo de índices de particiones
rng = np.random.default_rng(123)
B = 50  # aumentar para IC más estables (p. ej., 200-1000)
coef_matrix = []
n_parts = len(parts)

for b in range(B):
    sample_idx = rng.integers(0, n_parts, size=n_parts)  # remuestreo de particiones
    coef_b = fit_sgd_on_parts(sample_idx, seed=int(rng.integers(0,1e9)))
    coef_matrix.append(coef_b)

coef_matrix = np.vstack(coef_matrix)
coef_mean = coef_matrix.mean(axis=0)
coef_lo = np.quantile(coef_matrix, 0.025, axis=0)
coef_hi = np.quantile(coef_matrix, 0.975, axis=0)

# Reporta 10 primeros coeficientes con IC95%
print("\n[Logística-SGD] 10 coeficientes (media bootstrap e IC95%):")
for name, m, lo, hi in list(zip(feature_cols, coef_mean, coef_lo, coef_hi))[:10]:
    print(f"{name:25s}  β≈{m: .4f}  IC95% [{lo: .4f}, {hi: .4f}]")
```

***

Interpretación (Opción A)

1.  **Prueba t de Welch** (agregada):

*   Usa solo **n, media y varianza por grupo** → **no** cargas filas a memoria.
*   **p‑valor** < 0.05 + **IC95%** que **no** cruce 0 ⇒ diferencia estadísticamente significativa en el **ingreso promedio** entre tratado y control.
*   **Cohen’s d** resume el **tamaño del efecto** (relevancia práctica).

2.  **Modelos online (SGD)**:

*   **Regresión (ingreso)**: `R^2` en *hold‑out* indica proporción de varianza explicada.
*   **Logística (ticket alto)**: `AUC` evalúa poder discriminante global.
*   **Coeficientes**:
    *   En **logística**, el signo indica si incrementa/reduce la log‑odds; puedes aproximar **odds ratios** con `np.exp(beta)`.
    *   **Bootstrap de particiones** entrega **IC95%** empíricos de β sin requerir matrices completas; útil cuando el *solver* no expone varianzas.

3.  **Escalabilidad**:

*   El entrenamiento recorre **particiones** (mini‑batches distribuidos), evitando *OOM*.
*   El **bootstrap distribuido** remuestrea particiones (no filas), manteniendo bajo el tráfico al driver.

***

Opción B — PySpark (DataFrame API + MLlib)

> Útil si ya trabajas con un clúster Spark. Aquí mantenemos todo distribuido y usamos MLlib para los modelos.

```python
# ============================================
# 0) SparkSession
# ============================================
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.window import Window

spark = SparkSession.builder \
    .appName("Inferencia_y_Modelado_BigData") \
    .getOrCreate()

# ============================================
# 1) Lectura parquet y limpieza
# ============================================
df = spark.read.parquet("s3://mi-bucket/ventas/")  # o ruta HDFS/local
df = (df
      .select("tratamiento","ingreso","precio","publicidad","dia","sucursal")
      .where((F.col("ingreso") > 0) & (F.col("precio") > 0) & (F.col("publicidad") >= 0))
      .dropna())

# ============================================
# 2) Inferencia (Welch) con agregados distribuidos
# ============================================
stats_g = (df.groupBy("tratamiento")
    .agg(
        F.count("*").alias("n"),
        F.avg("ingreso").alias("mean"),
        F.variance("ingreso").alias("var")
    ))

row0 = stats_g.where("tratamiento=0").collect()[0]
row1 = stats_g.where("tratamiento=1").collect()[0]
n0,m0,v0 = row0["n"], row0["mean"], row0["var"]
n1,m1,v1 = row1["n"], row1["mean"], row1["var"]

import math
from scipy import stats as st

t_num = (m1 - m0)
t_den = math.sqrt(v1/n1 + v0/n0)
t_stat = t_num / t_den

df_num = (v1/n1 + v0/n0)**2
df_den = (v1**2 / (n1**2*(n1-1))) + (v0**2 / (n0**2*(n0-1)))
df_welch = df_num / df_den
p_val = 2*st.t.sf(abs(t_stat), df=df_welch)

print(f"[Spark] Welch t={t_stat:.3f}, gl={df_welch:.1f}, p={p_val:.3g}")

# ============================================
# 3) Feature engineering y ensamblado para MLlib
# ============================================
from pyspark.ml.feature import StringIndexer, OneHotEncoder, VectorAssembler, StandardScaler
from pyspark.ml import Pipeline

# Umbral P75 distribuido para target binario
p75 = df.approxQuantile("ingreso", [0.75], 1e-4)[0]
df_bin = df.withColumn("y_bin", (F.col("ingreso") >= F.lit(p75)).cast("int"))

# Indexado + one-hot
idx_dia = StringIndexer(inputCol="dia", outputCol="dia_idx", handleInvalid="keep")
idx_suc = StringIndexer(inputCol="sucursal", outputCol="suc_idx", handleInvalid="keep")
ohe = OneHotEncoder(inputCols=["dia_idx","suc_idx"], outputCols=["dia_ohe","suc_ohe"])

assembler = VectorAssembler(
    inputCols=["tratamiento","precio","publicidad","dia_ohe","suc_ohe"],
    outputCol="features_raw"
)
scaler = StandardScaler(inputCol="features_raw", outputCol="features", withMean=True, withStd=True)

pipe_prep = Pipeline(stages=[idx_dia, idx_suc, ohe, assembler, scaler])
prep_model = pipe_prep.fit(df_bin)
ds = prep_model.transform(df_bin).select("ingreso","y_bin","features")

# Split
train, test = ds.randomSplit([0.8, 0.2], seed=42)

# ============================================
# 4) Modelos MLlib: Regresión y Logística
# ============================================
from pyspark.ml.regression import LinearRegression
from pyspark.ml.classification import LogisticRegression
from pyspark.ml.evaluation import RegressionEvaluator, BinaryClassificationEvaluator

# --- Regresión lineal (solver 'normal' para p-val en algunos entornos) ---
lr = LinearRegression(featuresCol="features", labelCol="ingreso", predictionCol="pred")
lrm = lr.fit(train)
pred_r = lrm.transform(test)

r2 = RegressionEvaluator(labelCol="ingreso", predictionCol="pred", metricName="r2").evaluate(pred_r)
print("[Spark] R^2 test:", round(r2, 4))

# --- Logística ---
logit = LogisticRegression(featuresCol="features", labelCol="y_bin", probabilityCol="prob")
logm = logit.fit(train)
pred_c = logm.transform(test)

auc = BinaryClassificationEvaluator(labelCol="y_bin", rawPredictionCol="rawPrediction").evaluate(pred_c)
print("[Spark] AUC test:", round(auc, 4))

# Coeficientes y (opcional) odds ratios
coefs = logm.coefficients.toArray()
odds_ratios = np.exp(coefs)
print("[Spark] 5 OR primeros coeficientes:", odds_ratios[:5])

# Nota: p-valores de coeficientes no están disponibles universalmente en Logit MLlib.
# Para IC de coeficientes en Spark, usa bootstrap por muestreo estratificado + ajuste repetido.
```

Interpretación (Opción B)

*   **Welch**: igual que en Opción A, pero agregados y cuantil (P75) se calculan de forma **distribuida**.
*   **LinearRegression**: reporta `R^2` en *test*, coeficientes interpretables (MXN por unidad de *feature* estandarizada si aplicaste `StandardScaler`).
*   **LogisticRegression**: interpreta `exp(β)` como **odds ratio**; verifica `AUC` en *test*. Para IC, usa **bootstrap distribuido** (reentrenar sobre *samples* estratificados y colectar cuantiles de β).

***

Recomendaciones prácticas (Big Data)

1.  **Inferencia clásica**: calcula **agregados** (n, media, var) por grupo y aplica fórmulas analíticas (t de Welch, ICs) en el driver.
2.  **Modelos**:
    *   Si no tienes clúster, usa **aprendizaje incremental** (SGD) con **mini-lotes por partición** (Dask).
    *   Con clúster, usa **Spark MLlib** para lectura, *feature engineering* y ajuste distribuido.
3.  **IC de coeficientes**:
    *   Para modelos *online* sin varianzas analíticas, aplica **bootstrap** (remuestreo de particiones o estratos) y toma **percentiles** 2.5%–97.5%.
4.  **Validez y estabilidad**:
    *   Monitorea **drift**: recalcula cuantil P75 y re‑entrena periódicamente.
    *   Usa **AUC / R²** en *test* y **curvas de aprendizaje** por tamaño de muestra.
