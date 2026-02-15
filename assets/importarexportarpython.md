#### Importar y exportar datos desde Excel

Ejemplo 

*   Lectura desde Excel con control fino de columnas, tipos, fechas, valores perdidos, y conversores.
*   Escritura a Excel (múltiples hojas, sin índice, posiciones específicas).
*   **Anexado** a un archivo existente sin sobrescribir hojas.
*   Formateo (ancho de columna, formatos numéricos, estilo de encabezados, congelar paneles, validación y formato condicional) vía **openpyxl**.
*   Casos típicos: exportar pivot, exportar con fórmulas, y manejo de errores.

> **Requisitos**
>
> ```bash
> pip install pandas openpyxl
> # (Opcional para .xls antiguos) pip install xlrd==2.0.1
> ```
>
> *Nota:* `pandas` usa `openpyxl` como motor para `.xlsx`. Para `.xls` (legado) se requiere `xlrd` compatible.

***
Script autocontenido: Importar y exportar Excel con pandas + openpyxl

> **Estructura**:
>
> 1.  Crear un DataFrame de ejemplo (si no tienes el archivo)
> 2.  **Importar** desde Excel con diferentes opciones
> 3.  **Exportar** a Excel (crear archivo nuevo con múltiples hojas)
> 4.  **Anexar** a un Excel existente sin perder hojas
> 5.  **Formatear** el libro con openpyxl (fuentes, anchos, números, condicional)
> 6.  Buenas prácticas y manejo de errores

```python
import os
from datetime import datetime

import numpy as np
import pandas as pd

# ------------------------------------------------------------
# 1) Datos de ejemplo (si no cuentas con un Excel de entrada)
# ------------------------------------------------------------
np.random.seed(42)
n = 1000
df_ventas = pd.DataFrame({
    "fecha": pd.date_range("2025-12-01", periods=n, freq="D"),
    "sucursal": np.random.choice(["AGS-Centro", "AGS-Norte", "AGS-Sur"], size=n, p=[0.5, 0.3, 0.2]),
    "categoria": np.random.choice(["Electrónica", "Hogar", "Moda", "Alimentos"], size=n),
    "precio": np.random.gamma(2.0, 120.0, size=n).round(2),
    "unidades": np.random.randint(1, 8, size=n),
    "descuento_pct": np.random.choice([0, 5, 10, 15], size=n, p=[0.6, 0.2, 0.15, 0.05]),
})
df_ventas["importe_neto"] = df_ventas["precio"] * df_ventas["unidades"] * (1 - df_ventas["descuento_pct"]/100)

# Guardar un Excel “de entrada” para la demo
os.makedirs("data", exist_ok=True)
excel_in = "data/entrada.xlsx"
with pd.ExcelWriter(excel_in, engine="openpyxl") as xw:
    df_ventas.to_excel(xw, index=False, sheet_name="ventas")
    # Hoja adicional con metadatos
    pd.DataFrame({"parametro": ["version", "creado_en"], "valor": ["1.0", datetime.now().isoformat()]})\
      .to_excel(xw, index=False, sheet_name="metadata")

print(f"Archivo de entrada creado: {excel_in}")

# ------------------------------------------------------------
# 2) IMPORTAR DESDE EXCEL (lectura flexible)
# ------------------------------------------------------------

# 2.1) Lectura de una sola hoja con control de columnas y tipos
usecols = ["fecha", "sucursal", "categoria", "precio", "unidades", "descuento_pct", "importe_neto"]
dtype_map = {
    "sucursal": "category",
    "categoria": "category",
    "precio": "float64",
    "unidades": "Int64",         # enteros con soporte para NA
    "descuento_pct": "Int64",
    "importe_neto": "float64",
}
df = pd.read_excel(
    excel_in,
    sheet_name="ventas",
    usecols=usecols,
    dtype=dtype_map,
    engine="openpyxl",
    parse_dates=["fecha"],              # convierte a datetime
    na_values=["NA", "N/A", "", None],  # valores que tratar como NA
)

# 2.2) Lectura de múltiples hojas a la vez (dict de DataFrame)
all_sheets = pd.read_excel(excel_in, sheet_name=None, engine="openpyxl")
df_meta = all_sheets.get("metadata")

# 2.3) Lectura con conversores personalizados (p. ej., forzar moneda)
def parse_precio(x):
    # elimina posible símbolo y separador de miles
    if isinstance(x, str):
        x = x.replace("$", "").replace(",", "")
    try:
        return float(x)
    except Exception:
        return np.nan

df2 = pd.read_excel(
    excel_in,
    sheet_name="ventas",
    converters={"precio": parse_precio},  # aplica función a esa columna
    engine="openpyxl",
)

print("Lectura OK:", df.shape, " | Columnas:", df.dtypes.to_dict())

# ------------------------------------------------------------
# 3) EXPORTAR A EXCEL (crear libro con múltiples hojas)
# ------------------------------------------------------------
pivot_cat = (
    df.pivot_table(index="categoria", values="importe_neto", aggfunc=["sum", "mean", "count"])
      .round(2)
)
pivot_dia = (
    df.assign(dia=df["fecha"].dt.date)
      .groupby("dia", as_index=False)["importe_neto"].sum()
)

excel_out = "data/salida.xlsx"
with pd.ExcelWriter(excel_out, engine="openpyxl") as xw:
    # Hoja 1: datos crudos (muestra)
    df.head(500).to_excel(xw, index=False, sheet_name="muestra")

    # Hoja 2: agregados por categoría
    pivot_cat.to_excel(xw, sheet_name="resumen_categoria")

    # Hoja 3: agregados por día (con fecha como columna, sin índice)
    pivot_dia.to_excel(xw, index=False, sheet_name="resumen_diario", startrow=1, startcol=0)

    # Hoja 4: exportar con fórmula (ej.: margen = importe * 0.18)
    hoja = "con_formula"
    df_small = df[["fecha", "categoria", "importe_neto"]].head(20).copy()
    df_small.to_excel(xw, index=False, sheet_name=hoja)
    # Escribimos una fórmula Excel: en la columna D = C * 0.18
    ws = xw.sheets[hoja]
    ws["D1"] = "margen_est"
    for r in range(2, 2 + len(df_small)):
        ws[f"D{r}"] = f"=C{r}*0.18"

print(f"Archivo exportado: {excel_out}")

# ------------------------------------------------------------
# 4) ANEXAR A UN EXCEL EXISTENTE (sin sobrescribir)
# ------------------------------------------------------------
# Agregar una hoja nueva "parametros" a 'salida.xlsx'
with pd.ExcelWriter(excel_out, engine="openpyxl", mode="a", if_sheet_exists="new") as xw:
    params = pd.DataFrame({
        "clave": ["autor", "generado_en"],
        "valor": ["DIAZ EDGAR OSWALDO", datetime.now().strftime("%Y-%m-%d %H:%M")]
    })
    params.to_excel(xw, index=False, sheet_name="parametros")

# Anexar datos a una hoja existente (append “debajo”):
# Truco: leer hoja -> concatenar -> sobrescribir solo esa hoja.
# (openpyxl no soporta appends directos como CSV; esta es la forma robusta)
df_exist = pd.read_excel(excel_out, sheet_name="resumen_diario", engine="openpyxl")
nuevo_bloque = pd.DataFrame({"dia": [df["fecha"].max().date()], "importe_neto": [float(df["importe_neto"].tail(50).sum())]})
df_new = pd.concat([df_exist, nuevo_bloque], ignore_index=True)
with pd.ExcelWriter(excel_out, engine="openpyxl", mode="a", if_sheet_exists="replace") as xw:
    df_new.to_excel(xw, index=False, sheet_name="resumen_diario")

print("Anexado correcto en 'resumen_diario'.")

# ------------------------------------------------------------
# 5) FORMATEO CON openpyxl (columnas, estilos, formatos, validación)
# ------------------------------------------------------------
from openpyxl import load_workbook
from openpyxl.styles import Font, Alignment, PatternFill, Border, Side, numbers
from openpyxl.worksheet.datavalidation import DataValidation
from openpyxl.formatting.rule import ColorScaleRule

wb = load_workbook(excel_out)

# 5.1) Formatear encabezados y anchos en 'muestra'
ws = wb["muestra"]
# Encabezados en negrita y centrados
header_font = Font(bold=True, color="FFFFFF")
header_fill = PatternFill("solid", fgColor="4F81BD")  # azul
for cell in ws[1]:
    cell.font = header_font
    cell.alignment = Alignment(horizontal="center", vertical="center")
    cell.fill = header_fill

# Ajuste de ancho de columnas simple (basado en longitud máxima)
for col in ws.columns:
    max_len = 0
    col_letter = col[0].column_letter
    for cell in col[:200]:  # inspeccionar primeros valores para no tardar
        try:
            max_len = max(max_len, len(str(cell.value)))
        except Exception:
            pass
    ws.column_dimensions[col_letter].width = min(max(12, max_len + 2), 40)

# Congelar paneles (encabezado)
ws.freeze_panes = "A2"

# 5.2) Formato numérico en 'resumen_categoria': sum y mean como moneda
ws2 = wb["resumen_categoria"]
# Identifica columnas por nombre (en este DF MultiIndex, pandas pone niveles en filas 1..)
# Para simplificar, aplicamos a todas las celdas numéricas debajo de la fila 2
for row in ws2.iter_rows(min_row=3, min_col=2):
    for cell in row:
        if isinstance(cell.value, (int, float)):
            cell.number_format = numbers.FORMAT_CURRENCY_USD_SIMPLE  # "$#,##0.00"

# 5.3) Validación de datos (lista desplegable) en 'parametros'
if "parametros" in wb.sheetnames:
    wsp = wb["parametros"]
    dv = DataValidation(type="list", formula1='"DRAFT,FINAL"', allow_blank=False, showDropDown=True)
    dv.prompt = "Selecciona estado del reporte"
    dv.error = "Valor inválido. Usa DRAFT o FINAL."
    # Agregamos nueva columna de "estado"
    wsp["C1"] = "estado_reporte"
    wsp.add_data_validation(dv)
    dv.add("C2:C100")

# 5.4) Formato condicional (escala de color) en 'resumen_diario'
ws3 = wb["resumen_diario"]
# Encuentra última fila
last_row = ws3.max_row
# Aplica escala de color (verde–amarillo–rojo) a la columna 'importe_neto' (columna B)
rule = ColorScaleRule(start_type="min", start_color="63BE7B",
                      mid_type="percentile", mid_value=50, mid_color="FFEB84",
                      end_type="max", end_color="F8696B")
ws3.conditional_formatting.add(f"B2:B{last_row}", rule)

wb.save(excel_out)
print(f"Formateo aplicado a: {excel_out}")

# ------------------------------------------------------------
# 6) Buenas prácticas y manejo de errores (ejemplo simple)
# ------------------------------------------------------------
try:
    # Intento simple de leer una hoja que no existe
    pd.read_excel(excel_out, sheet_name="no_existe", engine="openpyxl")
except ValueError as e:
    print("[AVISO] Hoja 'no_existe' no encontrada. Detalle:", e)

print("\nProceso terminado con éxito.")
```

***

Explicación (qué hace cada bloque y por qué)

1.  **Lectura con control fino**
    *   `usecols` limita columnas; `dtype` y `parse_dates` garantizan **tipos consistentes**.
    *   `na_values` define qué cadenas se convierten en `NaN`.
    *   `converters` te deja **normalizar** formatos antes de tipificar (muy útil con moneda/porcentajes).

2.  **Escritura a múltiples hojas**
    *   `ExcelWriter(..., engine="openpyxl")` crea un libro nuevo.
    *   `to_excel(..., startrow=..., startcol=...)` ubica la tabla en una zona concreta (útil si deseas encabezados/portadas en filas 1–2).
    *   Puedes **incluir fórmulas** (ej., margen estimado) escribiendo en celdas con `xw.sheets[...]`.

3.  **Anexado a un Excel existente**
    *   Usa `mode="a"` para **agregar** contenido; `if_sheet_exists="new"` crea una hoja nueva si el nombre se repite.
    *   Para **agregar filas al final** de una hoja: lee la hoja, concatena y **reemplaza** solo esa hoja (`if_sheet_exists="replace"`). Es la forma más segura de evitar corrupción o duplicados.

4.  **Formateo con `openpyxl`**
    *   Encabezados con **negrita/centrado/color**.
    *   **Ancho de columnas** calculado (heurística).
    *   **Congelar paneles** para fijar encabezados.
    *   **Formatos numéricos** (`number_format`) para moneda, fecha, porcentaje, etc.
    *   **Validación** (listas desplegables) y **formato condicional** (escalas de color, reglas) para enriquecer tu reporte.

5.  **Buenas prácticas**
    *   Tipifica columnas (evitas sorpresas al analizar).
    *   Evita sobrescribir libros sin respaldo (usa `mode="a"` y nombres únicos de hoja).
    *   Para **archivos grandes**, considera **Parquet** como formato de trabajo y deja Excel para **entregar** resultados.
    *   Maneja errores comunes (hojas inexistentes, rutas, permisos).

***

## ➕ Variantes útiles

*   **Excel con múltiples DataFrames apilados**: usa `startrow` y `startcol` para colocar reportes en zonas distintas de la misma hoja.
*   **Estilos con `pandas` Styler**: prepara una tabla formateada (colores, barras) y luego exporta con `to_excel` (*soporte limitado*; para control fino, prefiere `openpyxl`).
*   **Automatización**: envuelve el flujo en una función que reciba ruta de entrada, lista de hojas, filtros, y ruta de salida; agenda con `cron`/Tareas programadas.
