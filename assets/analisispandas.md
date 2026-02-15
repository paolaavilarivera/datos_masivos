
***

Código de ejemplo (listo para ejecutar)

> **Supuesto de datos**: archivo CSV gigante con ventas (`fecha`, `categoria`, `precio`, `unidades`, `descuento_pct`, `sucursal`, `metodo_pago`).  
> **Objetivo**: KPI diarios y por categoría, estadística descriptiva robusta, y gráficos “amigables a big data”.

```python
# ============================================
# 0) Imports y configuración
# ============================================
import os
import math
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

plt.style.use("seaborn-v0_8-whitegrid")
plt.rcParams.update({"figure.figsize": (10, 5), "axes.titlesize": 14})

# ============================================
# 1) Parámetros de entrada / salida
# ============================================
CSV_PATH = "data/ventas_masivo.csv"   # <-- reemplaza por tu ruta
CHUNK_SIZE = 1_000_000                 # 1 millón de filas por trozo (ajusta según RAM/IO)
USECOLS = ["fecha","categoria","precio","unidades","descuento_pct","sucursal","metodo_pago"]

# Tipos eficientes (downcasting) para ahorrar memoria
DTYPE_MAP = {
    "categoria": "category",
    "sucursal": "category",
    "metodo_pago": "category",
    "precio": "float32",         # suficiente para montos típicos, reduce a la mitad de float64
    "unidades": "int16",         # si esperas valores < 32767
    "descuento_pct": "int8"      # 0..100 (o -128..127); ajusta si usas decimales
}

# ============================================
# 2) Acumuladores globales (agregados incrementales)
# ============================================
# - Ventas diarias (importe neto por día)
ventas_diarias = pd.Series(dtype="float64")  # índice: fecha; valor: importe_neto

# - Ventas por categoría (importe neto)
ventas_categoria = pd.Series(dtype="float64")  # índice: categoría; valor: importe_neto

# - Contadores básicos
filas_totales = 0

# - Estadística online (Welford) para precio y unidades
#   Mantiene: n, media, M2 (suma de cuadrados de desviaciones) sin almacenar todos los datos.
class OnlineStats:
    def __init__(self):
        self.n = 0
        self.mean = 0.0
        self.M2 = 0.0
    def update_batch(self, x: np.ndarray):
        # Variante vectorizada por lote
        x = x[~np.isnan(x)]
        if x.size == 0:
            return
        n_batch = x.size
        mean_batch = float(x.mean())
        M2_batch = float(((x - mean_batch)**2).sum())
        # combinación de estadísticos (ver fórmula de Welford agregada)
        delta = mean_batch - self.mean
        total_n = self.n + n_batch
        self.mean += delta * n_batch / total_n
        self.M2 += M2_batch + delta**2 * (self.n * n_batch / total_n)
        self.n = total_n
    def finalize(self):
        var = self.M2 / (self.n - 1) if self.n > 1 else float("nan")
        std = math.sqrt(var) if var == var else float("nan")
        return {"n": self.n, "mean": self.mean, "var": var, "std": std}

stats_precio = OnlineStats()
stats_unidades = OnlineStats()

# ============================================
# 3) Bucle de lectura por trozos (chunking)
# ============================================
read_iter = pd.read_csv(
    CSV_PATH,
    usecols=USECOLS,
    dtype=DTYPE_MAP,
    chunksize=CHUNK_SIZE,
    parse_dates=["fecha"],   # parsing perezoso por chunk
    infer_datetime_format=True
)

for i, chunk in enumerate(read_iter, start=1):
    # --- 3.1 Limpieza mínima y features ---
    # coerción segura
    chunk["precio"] = pd.to_numeric(chunk["precio"], errors="coerce")
    chunk["unidades"] = pd.to_numeric(chunk["unidades"], errors="coerce").astype("float32")
    chunk["descuento_pct"] = pd.to_numeric(chunk["descuento_pct"], errors="coerce")
    # Importe neto (precio * unidades * (1 - descuento/100))
    chunk["importe_neto"] = (chunk["precio"] * chunk["unidades"] * (1.0 - chunk["descuento_pct"]/100.0)).astype("float64")

    # Filtrado de valores imposibles
    chunk = chunk[(chunk["importe_neto"] >= 0) & (chunk["precio"] > 0) & (chunk["unidades"] > 0)]

    # --- 3.2 Agregación incremental ---
    # a) Diario
    diario = chunk.groupby(chunk["fecha"].dt.date, observed=True)["importe_neto"].sum()
    # combina con acumulador global alineando índices
    ventas_diarias = ventas_diarias.add(diario, fill_value=0.0)

    # b) Por categoría (¡ojo: categorías pueden ser muchas! luego recortamos a Top-K)
    por_cat = chunk.groupby("categoria", observed=True)["importe_neto"].sum()
    ventas_categoria = ventas_categoria.add(por_cat, fill_value=0.0)

    # --- 3.3 Estadística online (Welford) ---
    stats_precio.update_batch(chunk["precio"].to_numpy())
    stats_unidades.update_batch(chunk["unidades"].to_numpy())

    filas_totales += len(chunk)

    # Log ocasional
    if i % 5 == 0:
        print(f"[Chunk {i}] filas procesadas acumuladas: {filas_totales:,}")

# ============================================
# 4) Post-procesado de agregados
# ============================================
ventas_diarias.index = pd.to_datetime(ventas_diarias.index)   # garantizar datetime index
ventas_diarias = ventas_diarias.sort_index()
ventas_categoria = ventas_categoria.sort_values(ascending=False)

# Top-K categorías para visualización (reduce ruido)
TOP_K = 10
topk_cat = ventas_categoria.head(TOP_K)

# Estadísticos finales
precio_stats = stats_precio.finalize()
unidades_stats = stats_unidades.finalize()

print("\n=== Resumen estadístico (precio) ===")
print(precio_stats)
print("\n=== Resumen estadístico (unidades) ===")
print(unidades_stats)
print(f"\nFilas totales válidas: {filas_totales:,}")
print(f"Categorías distintas: {ventas_categoria.index.nunique()}")

# ============================================
# 5) Visualizaciones efectivas (sobre agregados)
# ============================================
# 5.1 Serie temporal de ventas diarias (suavizada con media móvil)
ventas_mm7 = ventas_diarias.rolling(window=7, min_periods=1).mean()

fig, ax = plt.subplots()
ax.plot(ventas_diarias.index, ventas_diarias.values, color="#457b9d", alpha=0.4, label="Diario")
ax.plot(ventas_mm7.index, ventas_mm7.values, color="#1d3557", linewidth=2.5, label="Media móvil 7d")
ax.set_title("Ventas netas diarias (con suavizado 7d)")
ax.set_xlabel("Fecha")
ax.set_ylabel("MXN")
ax.legend()
fig.tight_layout()
plt.show()

# 5.2 Barras Top-K categorías
fig, ax = plt.subplots()
topk_cat.sort_values().plot(kind="barh", ax=ax, color="#2a9d8f")
ax.set_title(f"Top-{TOP_K} categorías por venta neta")
ax.set_xlabel("MXN")
ax.set_ylabel("Categoría")
fig.tight_layout()
plt.show()

# 5.3 Histograma log-binned del importe diario (evita sesgos por colas largas)
#     Nota: al usar binning logarítmico evitamos gráficos “aplastados” por unos pocos días extremos.
x = ventas_diarias.values
x = x[x > 0]
bins = np.logspace(np.log10(x.min()), np.log10(x.max()), num=40)

fig, ax = plt.subplots()
ax.hist(x, bins=bins, color="#e76f51", alpha=0.85)
ax.set_xscale("log")
ax.set_title("Distribución de ventas diarias (binning log)")
ax.set_xlabel("MXN (escala log)")
ax.set_ylabel("Frecuencia")
fig.tight_layout()
plt.show()

# ============================================
# 6) Persistencia de resultados en formato eficiente (opcional)
# ============================================
# Guardar agregados a Parquet para análisis posteriores rápidos
out_dir = "out"
os.makedirs(out_dir, exist_ok=True)
ventas_diarias.rename("venta_diaria").to_frame().to_parquet(f"{out_dir}/ventas_diarias.parquet", index=True)
ventas_categoria.rename("venta_categoria").to_frame().to_parquet(f"{out_dir}/ventas_categoria.parquet", index=True)
```

***

Explicación detallada (por qué esto es “friendly” a datos masivos)

1) **Lectura por trozos (`chunksize`)**

*   En lugar de `read_csv` completo (que intenta cargar todo a RAM), usamos un **iterador de bloques** que procesa 1 millón de filas por pasada (ajústalo según tu máquina).
*   Cada **chunk** se limpia, se transforma y se **agrega** inmediatamente; **no** guardamos todos los datos crudos.

2) **Tipificación y *downcasting***

*   Las columnas numéricas se tipifican a `float32`/`int16`/`int8` cuando es posible: **mitad o menos** del consumo de memoria que `float64`/`int64`.
*   Las columnas categóricas (`category`) almacenan internamente **códigos** en lugar de strings repetidos, **drásticamente** más compacto.

3) **Agregación incremental**

*   `groupby(...).sum()` sobre cada trozo y después combinamos con `Series.add(..., fill_value=0.0)`.
*   Esto permite acumular **KPI diarios** y **por categoría** sin mantener todas las filas en memoria.
*   Para cardinalidades muy grandes (p. ej., miles de categorías), puedes:
    *   Guardar parciales por lote a disco y **reducir** después, o
    *   Quedarte sólo con **Top‑K** por trozo (con `nlargest`) antes de sumar al global.

4) **Estadística “online” (Welford)**

*   Calculamos `media`, `varianza`, `std` **sin** almacenar todas las observaciones: sólo mantenemos `n`, `mean` y `M2`.
*   Es **numéricamente estable** y lineal en memoria. Útil para KPIs globales de precisión *double* aunque los datos entren por lotes.

5) **Visualización efectiva para Big Data**

*   En lugar de graficar millones de puntos, graficamos **agregados** (línea diaria, barras Top‑K).
*   Usamos **binning logarítmico** para histogramas de colas largas, evitando gráficos “aplastados”.
*   Si necesitas dispersión de millones de puntos, considera **hexbin** o **agregación previa** por celdas/bins, y luego grafica el resultado agregado.

6) **Persistencia en Parquet**

*   Guardamos los resultados en **Parquet**, formato columnar comprimido, ideal para análisis futuros **rápidos** y **ligeros** con pandas (o motores distribuidos).

***

Variantes y extensiones

*   **Entrada Parquet**: si tus datos ya vienen en Parquet, reemplaza `read_csv` por `read_parquet` con `columns=[...]` (selección columnar muy rápida).
*   **Lectura desde S3/Azure/GCS**: pandas soporta rutas como `s3://bucket/file.csv` con los *fs* adecuados (`s3fs`, `gcsfs`, `adlfs`).
*   **Más velocidad en CSV**: fija `engine="pyarrow"` (si disponible) y `usecols` para reducir parsing.
*   **Muestreo estratificado** para EDA rápida: procesa 5–10% para explorar, fija el pipeline, luego corre al 100%.
*   **Distribuido**: cuando el archivo exceda lo razonable para *chunking*, evoluciona este patrón a **Dask** o **Spark** (la lógica de agregación y Welford se conserva).

