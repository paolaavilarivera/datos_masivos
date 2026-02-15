


1) Preparación del entorno y generación de datos

```python
# --- Librerías ---
import numpy as np
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

# Opcional: estilo gráfico
sns.set(style="whitegrid", context="talk")

# --- Reproducibilidad ---
rng = np.random.default_rng(42)

# --- Parámetros del dataset sintético ---
n = 15_000
fechas = pd.date_range("2025-01-01", periods=180, freq="D")
sucursales = ["AGS-Centro", "AGS-Norte", "AGS-Sur", "AGS-Altaria"]
categorias = ["Electrónica", "Hogar", "Moda", "Alimentos", "Deportes"]
metodos = ["Efectivo", "Tarjeta", "Wallet"]

# --- Generación de campos ---
df = pd.DataFrame({
    "fecha": rng.choice(fechas, size=n),
    "sucursal": rng.choice(sucursales, size=n, p=[0.35, 0.25, 0.25, 0.15]),
    "categoria": rng.choice(categorias, size=n, p=[0.25, 0.2, 0.2, 0.25, 0.1]),
    "precio_unitario": rng.normal(loc=350, scale=120, size=n).round(2).clip(20, 1500),
    "unidades": rng.integers(1, 6, size=n),
    "descuento_pct": rng.choice([0, 5, 10, 15, 20], size=n, p=[0.5, 0.2, 0.15, 0.1, 0.05]),
    "metodo_pago": rng.choice(metodos, size=n, p=[0.35, 0.55, 0.10]),
})

# Introducir valores faltantes/anómalos controlados (simulan problemas reales)
mask_missing = rng.choice([True, False], size=n, p=[0.02, 0.98])
df.loc[mask_missing, "precio_unitario"] = np.nan
mask_outliers = rng.choice([True, False], size=n, p=[0.01, 0.99])
df.loc[mask_outliers, "unidades"] = df.loc[mask_outliers, "unidades"] * 10  # outliers de cantidad

# Ingresos brutos y netos
df["importe_bruto"] = df["precio_unitario"] * df["unidades"]
df["importe_neto"] = df["importe_bruto"] * (1 - df["descuento_pct"]/100)
```

**Qué hicimos y por qué**

*   **Dataset sintético**: nos permite practicar sin exponer datos sensibles.
*   **Valores faltantes y outliers**: representan problemas comunes de calidad que debemos gestionar.
*   **Features calculadas**: `importe_bruto`/`importe_neto` habilitan análisis de negocio.

***

2) **Gestión de datos**: validación, tipificación, imputación y tratamiento de outliers

```python
# --- Tipificación adecuada ---
df["fecha"] = pd.to_datetime(df["fecha"], errors="coerce")
for col_cat in ["sucursal", "categoria", "metodo_pago"]:
    df[col_cat] = df[col_cat].astype("category")

# --- Reglas de negocio / validaciones ---
# 1) Precio unitario > 0 y razonable
# 2) Unidades en rango plausible
# 3) Fechas válidas dentro del rango esperado
valid_price = df["precio_unitario"].between(1, 5000) | df["precio_unitario"].isna()
valid_units = df["unidades"].between(1, 20)  # regla de negocio para tiendita
valid_date = df["fecha"].between(df["fecha"].min(), df["fecha"].max())

df["flag_validez"] = valid_price & valid_units & valid_date

# --- Imputación de faltantes ---
# Estrategia: imputar precio por mediana a nivel categoría (robusta a outliers)
mediana_cat = df.groupby("categoria")["precio_unitario"].transform("median")
df.loc[df["precio_unitario"].isna(), "precio_unitario"] = mediana_cat[df["precio_unitario"].isna()]

# Recalcular importes tras imputación
df["importe_bruto"] = df["precio_unitario"] * df["unidades"]
df["importe_neto"] = df["importe_bruto"] * (1 - df["descuento_pct"]/100)

# --- Tratamiento de outliers en 'unidades' ---
q1, q3 = df["unidades"].quantile([0.25, 0.75])
iqr = q3 - q1
upper_bound = q3 + 1.5 * iqr

# Estrategia: winsorización suave
df["unidades_adj"] = np.where(df["unidades"] > upper_bound, np.floor(upper_bound).astype(int), df["unidades"])
df["importe_neto_adj"] = df["precio_unitario"] * df["unidades_adj"] * (1 - df["descuento_pct"]/100)
```

**Claves de gestión**

*   **Tipificación**: `category` y `datetime` mejoran eficiencia y evitan errores.
*   **Imputación robusta**: la **mediana por categoría** reduce sesgo frente a valores extremos.
*   **Outliers**: **winsorización** evita que pocos casos distorsionen conclusiones, respetando la señal.

***

3) **Exploración de datos (EDA)**: entendimiento estructurado

```python
# Resumen rápido
resumen = df[["precio_unitario","unidades","descuento_pct","importe_neto_adj"]].describe()
print(resumen)

# Análisis categórico
ventas_por_categoria = (
    df.groupby("categoria", observed=True)["importe_neto_adj"]
      .sum()
      .sort_values(ascending=False)
)
print(ventas_por_categoria)

# Series temporales: ventas diarias
diario = (df
          .set_index("fecha")
          .groupby(pd.Grouper(freq="D"))["importe_neto_adj"]
          .sum()
          .rename("venta_diaria"))
print(diario.head())
```

**Qué buscamos en EDA**

*   **Distribuciones** (tendencia central, dispersión, asimetrías).
*   **Relaciones** (por categoría, sucursal, método de pago).
*   **Patrones temporales** (estacionalidad, picos, valles).
*   **Calidad** (proporción de registros válidos, faltantes, inconsistencias).

***

4) **Visualización efectiva**: comunicar hallazgos con intención

> Reglas prácticas:
>
> *   Emplea títulos **específicos** que respondan *qué* y *para qué*.
> *   Elige **marcas** y **escalas** que faciliten comparación (barras para totales, líneas para tiempo, caja para dispersión).
> *   Añade **anotaciones** cuando haya puntos de interés (picos/caídas).

> **Nota**: No escribas código dentro de tablas; los siguientes bloques son independientes.

4.1. Distribución de precio y detección de colas (histograma + KDE)

```python
plt.figure(figsize=(9,5))
sns.histplot(df["precio_unitario"], bins=40, kde=True, color="#2a9d8f")
plt.title("Distribución del precio unitario (post-imputación)")
plt.xlabel("Precio unitario")
plt.ylabel("Frecuencia")
plt.tight_layout()
plt.show()
```

**Interpretación**: visualiza si hay **colas pesadas** o concentraciones. Una cola derecha marcada sugiere presentes artículos *premium*; esto impacta políticas de descuento y *mix* de productos.

***

4.2. Impacto de outliers y control de dispersiones (boxplot)

```python
plt.figure(figsize=(9,5))
sns.boxplot(data=df, x="categoria", y="unidades", palette="Set2")
plt.title("Dispersión de unidades por categoría (antes del ajuste)")
plt.xlabel("Categoría")
plt.ylabel("Unidades por ticket")
plt.xticks(rotation=15)
plt.tight_layout()
plt.show()
```

**Interpretación**: identifica categorías con **tickets atípicos** (ej., *Electrónica* puede tener unidades bajas pero alto importe; *Alimentos*, más unidades por ticket).

***

4.3. Ventas por categoría (barras ordenadas)

```python
top_cat = ventas_por_categoria.reset_index().rename(columns={"importe_neto_adj":"venta"})
plt.figure(figsize=(9,5))
sns.barplot(data=top_cat, x="categoria", y="venta", order=top_cat["categoria"], color="#264653")
plt.title("Ventas netas ajustadas por categoría (ordenadas)")
plt.xlabel("Categoría")
plt.ylabel("MXN")
plt.xticks(rotation=15)
plt.tight_layout()
plt.show()
```

**Interpretación**: comunica **prioridades comerciales**. Las primeras categorías merecen foco en inventario, abastecimiento y promociones; las últimas pueden requerir revisión de precios o *merchandising*.

***

4.4. Tendencia temporal (línea diaria)

```python
plt.figure(figsize=(12,5))
sns.lineplot(x=diario.index, y=diario.values, color="#1d3557")
plt.title("Serie temporal de ventas diarias (importe neto ajustado)")
plt.xlabel("Fecha")
plt.ylabel("Venta diaria (MXN)")
plt.tight_layout()
plt.show()
```

**Interpretación**: observa **estacionalidad** (picos fin de semana / quincena) y **anomalías** (días con caída). Útil para planear personal, inventario y campañas.

***

4.5. Relación precio–unidades–descuento (dispersión con codificación por color/tamaños)

```python
plt.figure(figsize=(9,6))
sample = df.sample(2000, random_state=42)
sns.scatterplot(
    data=sample,
    x="precio_unitario", y="unidades_adj",
    hue="descuento_pct", palette="viridis", size="descuento_pct", sizes=(20, 200), alpha=0.7
)
plt.title("Precio vs Unidades (color/tamaño = descuento)")
plt.xlabel("Precio unitario (MXN)")
plt.ylabel("Unidades (ajustadas)")
plt.legend(bbox_to_anchor=(1.02, 1), borderaxespad=0)
plt.tight_layout()
plt.show()
```

**Interpretación**: permite ver si **mayor descuento** impulsa **más unidades** sólo en ciertos rangos de precio; clave para **elasticidad** y diseño de promociones.

***

Conclusiones y recomendaciones

1.  **Gestión**:
    *   Tipificar correctamente (`datetime`, `category`) y **documentar reglas de negocio** (rangos válidos, campos obligatorios).
    *   **Imputar** con estrategias robustas (mediana por grupo) y **controlar outliers** (winsorización o modelos robustos).
2.  **Exploración**:
    *   Combina **resúmenes numéricos** con cortes por **dimensiones de negocio** (categoría, sucursal, método de pago).
    *   Construye **series temporales** para detectar estacionalidad y eventos.
3.  **Visualización**:
    *   Usa el **gráfico apropiado** para cada pregunta (distribución, comparación, tendencias, relación).
    *   Titula con **intención** y agrega **anotaciones** para destacar hallazgos accionables.

***

Extensiones sugeridas

*   **Dashboard interactivo** (Streamlit/Plotly Dash) para seguimiento de KPIs.
*   **Modelos** de demanda (ARIMA/Prophet/XGBoost) usando la serie `diario`.
*   **Segmentación de tickets** (clustering) por mix de categorías y elasticidad al descuento.

