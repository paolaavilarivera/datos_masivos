#### Visualización dinámica

Requisitos

> Si usas un entorno nuevo, instala dependencias (una sola vez):

```bash
pip install "dask[complete]" hvplot holoviews datashader bokeh panel colorcet pyarrow
```

***

Código (guárdalo como `dashboard_bigdata.py`)

```python
# -*- coding: utf-8 -*-
"""
Dashboard interactivo para visualización dinámica de datos masivos con Dask + Datashader + hvPlot + Panel.

Cómo ejecutar:
    panel serve dashboard_bigdata.py --autoreload --show
"""

# ======================
# 1) IMPORTACIONES
# ======================
import dask.dataframe as dd
import dask
from dask import datasets as dask_datasets

import hvplot.dask  # activa hvplot para Dask
import hvplot.pandas
import holoviews as hv
import panel as pn
import colorcet as cc

hv.extension('bokeh')
pn.extension('tabulator')  # opcional para tablas

# ======================
# 2) PARÁMETROS GLOBALES
# ======================
# Rango temporal del dataset sintético (ajusta para reducir/aumentar tamaño)
START = "2024-01-01"
END   = "2024-12-31"
FREQ  = "1s"            # 1 segundo ⇒ ~31M de filas/año (demostración de escala)
PARTITION = "1d"        # particiones de 1 día en Dask (balance entre paralelismo y tamaño de tarea)

# ======================
# 3) CARGA / GENERACIÓN DE DATOS MASIVOS
# ======================
# Usamos el generador de Dask: timeseries => columnas: ['id','name','x','y','ts']
# - 'x' y 'y' se usan como coordenadas
# - 'ts' es marca de tiempo
# - 'name' es categórica con pocos valores (útil para filtrar)
# *Este dataset se genera de forma perezosa (lazy), sin cargar todo en memoria.

ddf = dask_datasets.timeseries(
    start=START,
    end=END,
    freq=FREQ,
    partition_freq=PARTITION,
    seed=42
).persist()  # persist opcional: fija en memoria distribuida si tienes clúster; en local, puedes quitarlo

# ======================
# 4) WIDGETS (FILTROS)
# ======================
# Obtenemos valores para widgets (computaciones pequeñas)
start_ts = ddf['ts'].min().compute()
end_ts   = ddf['ts'].max().compute()
names    = list(ddf['name'].unique().compute())

# Widgets
w_fecha  = pn.widgets.DateRangeSlider(
    name='Rango de fechas',
    start=start_ts, end=end_ts,
    value=(start_ts, end_ts),
    step=24*60*60*1000  # paso en ms (~1 día) para una mejor experiencia
)

w_nombres = pn.widgets.MultiChoice(
    name='Categorías (name)',
    options=sorted(names),
    value=sorted(names),  # por defecto: todas
    solid=False
)

# ======================
# 5) PIPELINE REACTIVO CON .interactive
# ======================
idf = ddf.interactive()

# Filtros
filtro_fecha = (idf.ts >= w_fecha.param.value_start) & (idf.ts <= w_fecha.param.value_end)
# Si no hay selección, mantenemos todas las categorías
filtro_nombres = idf.name.isin(w_nombres)

idf_filtrado = idf[filtro_fecha & filtro_nombres]

# ======================
# 6) GRÁFICOS (Datashader para “big data”)
# ======================
# 6.1 Nube de puntos masiva (x vs y) con Datashader
#     - datashade=True activa el pipeline de Datashader (agrega por pixel).
#     - dynspread ayuda a “expandir” puntos densos para visibilidad.
scatter_ds = idf_filtrado.hvplot.scatter(
    x='x', y='y',
    width=900, height=500,
    title='Distribución masiva de puntos (Datashader)',
    cmap=cc.fire,
    datashade=True,
    dynspread=True,
)

# 6.2 Serie de tiempo agregada con Datashader (línea)
#     - datashade=True evita dibujar millones de segmentos.
#     - 'aggregator' controla el resumen; 'mean' como ejemplo.
ts_line = idf_filtrado.hvplot.line(
    x='ts', y='y',
    width=900, height=250,
    title='Serie temporal agregada (Datashader)',
    datashade=True,
)

# ======================
# 7) LAYOUT DEL DASHBOARD
# ======================
intro = pn.pane.Markdown(
    """
# Visualización dinámica de datos masivos

**Stack:** Dask + Datashader + hvPlot + Panel  
- Acércate con *zoom* y desplázate (*pan*) para explorar densidades.
- Ajusta el rango de fechas y las categorías.
- Los gráficos usan **agregación a nivel de píxel** (Datashader), ideal para millones de puntos.

**Notas didácticas:**
- Evitamos el *overplotting* (solapamiento) en nubes enormes.
- El cálculo es *lazy* y paralelo (Dask).
- Datashader renderiza por píxeles ⇒ escalabilidad real.
""",
    sizing_mode='stretch_width'
)

controles = pn.Card(
    pn.Row(
        pn.Column("## Filtros", w_fecha, w_nombres, width=400),
    ),
    title="Panel de control",
    collapsed=False
)

layout = pn.Template(
    title='Big Data & Data — Visualización dinámica',
    main=[
        intro,
        controles,
        scatter_ds,
        ts_line,
    ]
)

# ======================
# 8) SERVIDOR PANEL
# ======================
# Esto permite: `panel serve dashboard_bigdata.py --autoreload --show`
def main():
    return layout

# Para ejecución con `panel serve`
pn.state.template.param.update(title='Big Data & Data — Visualización dinámica')
layout.servable()
```

***

¿Cómo ejecutar?

1.  Guarda el script como `dashboard_bigdata.py`.
2.  Instala dependencias (una vez):  
    `pip install "dask[complete]" hvplot holoviews datashader bokeh panel colorcet pyarrow`
3.  Lanza el servidor:
    ```bash
    panel serve dashboard_bigdata.py --autoreload --show
    ```
4.  Se abrirá en tu navegador un dashboard interactivo. Prueba hacer **zoom/pan** en la nube de puntos y mover los **sliders**.

***

Buenas prácticas para “datos masivos”

*   **Formato de entrada**: usa **Parquet** (columnar, comprimido) en vez de CSV cuando trabajes con datos reales.
*   **Lectura con Dask**: `dd.read_parquet("ruta/*.parquet")` aprovecha múltiples cores/particiones.
*   **Tipa bien**: reduce `float64`→`float32` cuando sea posible; categoriza strings (`category`).
*   **Datashader**:
    *   `datashade=True` para puntos/series densas.
    *   `dynspread=True` mejora la visibilidad de clusters densos.
    *   Prueba paletas de `colorcet` (`cc.fire`, `cc.blues`, etc.) para contrastar densidades.
*   **Interacción**: usa **Panel** (`pn.widgets`) para filtros; `.interactive` simplifica el enlace datos↔widgets.
*   **Escalabilidad**: en un clúster, inicializa un `Client` de Dask para paralelizar en serio.

***




