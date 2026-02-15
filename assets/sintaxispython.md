

#### Sintaxis y semántica de constructos esenciales de Python

| Constructo                               | Sintaxis (descriptiva)                              | Semántica (qué hace)                                                                                    | Notas / Errores comunes                                                                                                     |
| ---------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| **Asignación**                           | `nombre = expresión`                                | Vincula el resultado de una expresión a un identificador (o tupla de ellos).                            | La asignación es por **referencia** a objetos; usar `:=` (operador walrus) solo dentro de expresiones cuando sea apropiado. |
| **Asignación múltiple / desempaquetado** | `a, b = iterable_de_longitud_2`                     | Desempaqueta elementos en variables correspondientes.                                                   | Coincidencia estricta de longitudes; usar `*rest` para capturar remanentes.                                                 |
| **Anotaciones de tipo**                  | `nombre: Tipo = valor`                              | Asocia **metadatos de tipo**; no impone chequeos en tiempo de ejecución por defecto.                    | Usa herramientas como `mypy/pyright`; recuerda que son *hints*.                                                             |
| **Expresiones aritméticas / booleanas**  | `+, -, *, /, //, %, **, and, or, not`               | Evalúan operaciones numéricas o lógicas con precedencia definida.                                       | `and`/`or` retornan operandos, no necesariamente `True/False`; división `/` es flotante.                                    |
| **Comparaciones encadenadas**            | `a < b <= c`                                        | Evalúa pares en cadena con cortocircuito.                                                               | Más legible que `a<b and b<=c`.                                                                                             |
| **Estructuras de decisión**              | `if`, `elif`, `else`                                | Selecciona ramas según la verdad de expresiones.                                                        | “Verdad” en Python depende de *truthiness* (listas vacías → `False`, etc.).                                                 |
| **Bucles**                               | `for elemento in iterable:` … `while condición:` …  | Itera secuencialmente (`for`) o mientras una condición sea verdadera (`while`).                         | `for` usa el **protocolo de iterador**; evitar mutar contenedores durante iteración.                                        |
| **Control de bucle**                     | `break`, `continue`, `else` (en bucles)             | `break` sale del bucle; `continue` salta a la siguiente iteración; `else` corre si **no** hubo `break`. | El `else` en bucles se usa poco; útil con búsquedas exhaustivas.                                                            |
| **Funciones**                            | `def nombre(parámetros):` bloque                    | Define una función; parámetros posicionales, nombrados, `*args`, `**kwargs`, valores por defecto.       | Los **valores por defecto** se evalúan una vez (cuidado con mutables).                                                      |
| **Funciones lambda**                     | `lambda args: expr`                                 | Función anónima de una sola expresión.                                                                  | Limitadas a una expresión; prioriza `def` por legibilidad.                                                                  |
| **Retorno y no retorno**                 | `return expr` / sin `return`                        | Devuelve un valor; sin `return` devuelve `None`.                                                        | Evitar múltiples returns dispersos que afecten legibilidad.                                                                 |
| **Ámbitos**                              | `global`, `nonlocal`                                | Declaran que un nombre se resuelve en ámbito global o no local (en clausuras).                          | Úsalos con moderación; preferir pasar parámetros.                                                                           |
| **Comprehensions**                       | `[expr for x in it if cond]`, `{…}`, `{k:v …}`      | Construyen listas, sets o dicts de forma declarativa y perezosa (con `()` → generator).                 | Evitar anidamientos muy profundos; cuidar coste en memoria.                                                                 |
| **Slices**                               | `obj[inicio:fin:paso]`                              | Devuelve una vista/secuencia rebanada (según tipo).                                                     | Índices negativos y `None` son válidos; `fin` es **exclusivo**.                                                             |
| **Gestión de contexto**                  | `with gestor as nombre:`                            | Asegura adquisición/liberación de recursos (`__enter__`/`__exit__`).                                    | Fundamental para archivos, locks, sesiones.                                                                                 |
| **Excepciones**                          | `try/except/else/finally`, `raise`                  | Maneja condiciones excepcionales; `else` si no hubo excepción; `finally` siempre se ejecuta.            | Capturar solo excepciones esperadas; no uses `except:` desnudo.                                                             |
| **Clases**                               | `class Nombre(Base):` bloque                        | Define tipos personalizados; atributos y métodos; herencia única o múltiple.                            | Implementa `__init__`, `__repr__`, `__eq__` según necesidad; considera `@dataclass`.                                        |
| **Métodos especiales**                   | `__init__`, `__repr__`, `__len__`, etc.             | Integran el objeto con protocolos del lenguaje (secuencias, iterables, operadores).                     | Implementa solo los necesarios; mantener invariantes de clase.                                                              |
| **Decoradores**                          | `@decorador` sobre `def/class`                      | Transforma funciones/clases en el momento de la definición.                                             | Preservar metadatos con `functools.wraps`.                                                                                  |
| **Generadores**                          | `def f(): yield valor`                              | Funciones que producen secuencias perezosas; mantienen estado entre `yield`.                            | No mezclar retornos con datos y control inadvertidamente; `return x` levanta `StopIteration(x)`.                            |
| **Iteradores / protocolo**               | `iter(obj)`, `next(it)`                             | Define cómo iterar (`__iter__`, `__next__`).                                                            | Asegura `StopIteration` adecuado.                                                                                           |
| **Importación de módulos**               | `import m`, `from m import x as y`                  | Carga módulos/atributos en el espacio de nombres.                                                       | Evitar `from m import *`; gestionar `PYTHONPATH`/paquetes.                                                                  |
| **F‑strings**                            | `f"texto {expr:formato}"`                           | Interpolación y formato en tiempo de ejecución.                                                         | Cuidar expresiones con efectos; especificar formatos (`:.2f`, etc.).                                                        |
| **Pattern Matching** (3.10+)             | `match valor: case patrón:`                         | Desestructuración y ramificación por patrones estructurales.                                            | Añadir caso por defecto (`case _:`) para exhaustividad.                                                                     |
| **Anotaciones / Docstrings**             | `"""docstring"""` al inicio de módulo/función/clase | Documentación accesible vía `.__doc__`.                                                                 | Mantener actualizadas; siguen convención reST/Google/NumPy.                                                                 |

***

### Ejemplos breves (fuera de la tabla)

```python
# Asignación y desempaquetado
x, y = (10, 20)

# Función con tipos, defaults y *args/**kwargs
def area_rect(a: float, b: float = 1.0, *extras, **meta) -> float:
    return a * b

# Comprensión y generator
cuadrados = [n*n for n in range(5) if n % 2 == 0]
pares = (n for n in range(10) if n % 2 == 0)

# Gestión de contexto
with open("datos.csv", "r", encoding="utf-8") as f:
    primera = f.readline()

# Excepciones
try:
    valor = int("42")
except ValueError as e:
    print("Conversión inválida:", e)
else:
    print("Ok")
finally:
    print("Siempre se ejecuta")

# Clase con método especial
from dataclasses import dataclass
@dataclass
class Punto:
    x: float; y: float
    def __repr__(self): return f"Punto({self.x}, {self.y})"

# Pattern matching
def clasificar(t):
    match t:
        case (x, y) if x == y: return "diagonal"
        case (x, y): return "punto"
```

***

