#### Arrays & Dataframes

Integra una lógica típica de análisis Big Data (agregación y métricas)

Ejemplo: Procesamiento de Grandes Volúmenes de Datos con Arrays y DataFrames

Escenario

Simulación de 10 millones de registros de transacciones (caso típico: telecomunicaciones, sensores IoT, comercio electrónico o microdatos anonimizados).

---

Uso de Arrays (NumPy) para generación eficiente de datos

```python
import numpy as np
import pandas as pd
import time

# Simulación de gran volumen de datos
n = 10_000_000  # 10 millones de registros

start = time.time()

# Arrays NumPy (más eficientes que listas)
transaction_ids = np.arange(n)
user_ids = np.random.randint(1, 1_000_000, size=n)
amounts = np.random.normal(loc=500, scale=100, size=n)
categories = np.random.randint(1, 20, size=n)

end = time.time()
print(f"Tiempo de generación con NumPy: {end - start:.2f} segundos")
```

---

Creación de DataFrame para análisis estructurado

```python
start = time.time()

df = pd.DataFrame({
    "transaction_id": transaction_ids,
    "user_id": user_ids,
    "amount": amounts,
    "category": categories
})

end = time.time()
print(f"Tiempo de creación del DataFrame: {end - start:.2f} segundos")

print(df.head())
```


Operaciones Big Data (agregaciones masivas)

```python
start = time.time()

# Total por categoría
category_summary = df.groupby("category")["amount"].agg(["count", "mean", "sum"])

end = time.time()
print(f"Tiempo de agregación: {end - start:.2f} segundos")

print(category_summary)
```

