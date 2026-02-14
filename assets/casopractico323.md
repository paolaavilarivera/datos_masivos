#### Caso de Uso: Análisis de Sentimiento para evaluar la percepción social de una política pública
Nota: El análisis de sentimiento en redes sociales ofrece beneficios clave para proyectos de Big Data, pues permite comprender de manera inmediata las percepciones, emociones y opiniones de grandes comunidades. Al aprovechar el volumen, velocidad y variedad de datos generados por los usuarios, es posible identificar tendencias emergentes, evaluar la reacción ante eventos o productos y detectar riesgos reputacionales en tiempo real. Además, el uso de técnicas avanzadas de procesamiento de lenguaje natural facilita extraer patrones de comportamiento y generar información accionable para la toma de decisiones estratégicas. Esto convierte a las redes sociales en una fuente valiosa de insights para ciencia de datos.

Una institución pública desea evaluar la percepción ciudadana sobre una nueva política energética anunciada recientemente.

La red social **X** representa una fuente de datos no estructurados de alto valor para:

* Medir polarización social
* Detectar tendencias emergentes
* Identificar actores influyentes
* Analizar cambios temporales en el sentimiento

---

Enfoque Big Data

Desde la perspectiva de Big Data:

| Dimensión | Aplicación en el caso              |
| --------- | ---------------------------------- |
| Volumen   | Miles/millones de publicaciones    |
| Velocidad | Flujo en tiempo real               |
| Variedad  | Texto, hashtags, emojis, metadatos |
| Veracidad | Ruido, bots, sarcasmo              |
| Valor     | Inteligencia estratégica           |

El pipeline analítico incluirá:

1. Ingesta de datos vía API
2. Preprocesamiento NLP
3. Modelado de sentimiento
4. Visualización y análisis temporal
5. Interpretación estratégica

---

Arquitectura Analítica

```
API X → Data Collection → Cleaning → Tokenization → Sentiment Model → Aggregation → Dashboard
```

En R trabajaremos con:

* `rtweet` (conexión API)
* `tidytext`
* `dplyr`
* `ggplot2`
* `textclean`
* `syuzhet` (alternativa lexicográfica)

---

Extracción de Datos desde X en R

Nota: Desde 2024 la API de X tiene restricciones de pago. Se requiere cuenta desarrollador.

```r
Instalación 
install.packages(c("rtweet","tidyverse","tidytext","textclean","syuzhet"))

library(rtweet)
library(tidyverse)
library(tidytext)
library(textclean)
library(syuzhet)
```

Autenticación API

```r
token <- create_token(
  app = "nombre_app",
  consumer_key = "API_KEY",
  consumer_secret = "API_SECRET",
  access_token = "ACCESS_TOKEN",
  access_secret = "ACCESS_SECRET"
)
```

Extracción de Tweets

```r
tweets <- search_tweets(
  q = "reforma energetica",
  n = 5000,
  lang = "es",
  include_rts = FALSE
)

head(tweets$text)
```

---

Preprocesamiento NLP

El NLP (Procesamiento de Lenguaje Natural) es un conjunto de técnicas que permiten a las computadoras comprender, interpretar y analizar texto humano. En el análisis de sentimiento, el NLP identifica emociones, opiniones y polaridad en mensajes, facilitando transformar grandes volúmenes de texto en información útil y accionable para la toma de decisiones.

El texto en X contiene:

* URLs
* Menciones (@usuario)
* Hashtags
* Emojis
* Stopwords
* Ruido sintáctico

Limpieza

```r
tweets_clean <- tweets %>%
  select(status_id, created_at, text) %>%
  mutate(text = replace_url(text, ""),
         text = replace_tag(text, ""),
         text = replace_emoji(text),
         text = tolower(text))
```

---

Tokenización

La tokenización es el proceso de dividir un texto en unidades pequeñas llamadas tokens, como palabras o frases. En el análisis de sentimiento, permite estructurar el lenguaje natural para que los algoritmos identifiquen significado, patrones y emociones, facilitando evaluar la polaridad y clasificar opiniones de manera eficiente.

```r
tweets_tokens <- tweets_clean %>%
  unnest_tokens(word, text)
```

Eliminar stopwords en español:

Las stopwords son palabras comunes como “el”, “de” o “y” que no aportan significado relevante al análisis. En el análisis de sentimiento de una red social, se eliminan para reducir ruido, mejorar la calidad del procesamiento y permitir que los algoritmos se enfoquen en términos verdaderamente útiles para detectar emociones y opiniones.

```r
data("stop_words")

stop_words_es <- stop_words %>%
  filter(lexicon == "snowball")

tweets_tokens <- tweets_tokens %>%
  anti_join(stop_words_es, by = "word")
```

---

Análisis de Sentimiento Lexicográfico

El análisis de sentimiento lexicográfico utiliza diccionarios de palabras con valores asociados de polaridad y emoción para evaluar textos. Cada término aporta una puntuación que, al sumarse, permite determinar si el sentimiento general es positivo, negativo o neutro, ofreciendo una forma sencilla y transparente de interpretar opiniones.

Utilizaremos el lexicón **Bing** (positivo/negativo).

```r
bing_sentiment <- get_sentiments("bing")

tweets_sentiment <- tweets_tokens %>%
  inner_join(bing_sentiment, by = "word") %>%
  count(status_id, sentiment) %>%
  pivot_wider(names_from = sentiment,
              values_from = n,
              values_fill = 0) %>%
  mutate(score = positive - negative)
```

---

Agregación Temporal

La agregación temporal permite organizar el sentimiento de publicaciones en redes sociales según intervalos de tiempo, revelando patrones, tendencias y cambios emocionales. Esto facilita detectar eventos clave, medir su impacto, reducir ruido individual y generar indicadores más estables, mejorando el análisis y la toma de decisiones basada en comportamiento colectivo.

```r
tweets_analysis <- tweets_clean %>%
  left_join(tweets_sentiment, by = "status_id") %>%
  mutate(score = ifelse(is.na(score), 0, score),
         date = as.Date(created_at)) %>%
  group_by(date) %>%
  summarise(avg_sentiment = mean(score))
```

---

Visualización

La visualización de grandes volúmenes de datos en el análisis de sentimiento de una red social permite identificar tendencias, patrones y anomalías de forma clara e inmediata. Facilita comprender cambios en la opinión pública, comparar grupos y detectar eventos relevantes, convirtiendo datos complejos en información accesible para decisiones estratégicas.

```r
ggplot(tweets_analysis, aes(x = date, y = avg_sentiment)) +
  geom_line(color = "blue") +
  theme_minimal() +
  labs(title = "Evolución del Sentimiento en X",
       y = "Sentimiento Promedio",
       x = "Fecha")
```

Interpretación:

* Valores positivos → percepción favorable
* Valores negativos → percepción adversa
* Picos abruptos → eventos coyunturales

---

Alternativa: Modelo con `syuzhet`

El modelo con syuzhet utiliza diccionarios especializados para extraer sentimientos de textos mediante distintas metodologías, como NRC, Bing y Afinn. Convierte el contenido en valores emocionales y de polaridad, permitiendo analizar patrones afectivos. Es ampliamente usado por su precisión, flexibilidad y facilidad para comparar enfoques lexicográficos.

| **Característica**         | **NRC**                                                                                                        | **Bing**                                    |
| -------------------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| **Tipo de análisis**       | Emociones y polaridad                                                                                          | Solo polaridad                              |
| **Categorías**             | 8 emociones (alegría, tristeza, enojo, miedo, sorpresa, disgusto, confianza, anticipación) + positivo/negativo | Positivo y negativo                         |
| **Nivel de detalle**       | Alto, análisis afectivo profundo                                                                               | Medio, análisis rápido y directo            |
| **Tamaño del diccionario** | Más amplio y complejo                                                                                          | Más compacto y simple                       |
| **Uso ideal**              | Estudios que requieren distinguir emociones específicas                                                        | Evaluaciones rápidas de sentimiento general |


```r
sentiment_values <- get_sentiment(tweets_clean$text, method = "syuzhet")

tweets_clean$sentiment <- sentiment_values

mean(sentiment_values)
```

---

Extensión: Machine Learning Supervisado

El machine learning supervisado en el análisis de sentimiento utiliza modelos entrenados con textos previamente etiquetados como positivos, negativos o neutros. Permite aprender patrones lingüísticos y predecir el sentimiento de nuevas publicaciones en redes sociales con alta precisión, mejorando la clasificación automática y la interpretación de opiniones a gran escala.

Para un enfoque más avanzado:

1. Crear dataset etiquetado manualmente
2. Vectorización TF-IDF
3. Modelo:

   * Naive Bayes
   * SVM
   * Random Forest
   * BERT (vía `reticulate`)


| **Algoritmo**               | **Tipo**                                  | **Cómo funciona (resumen)**                                                                                                                                                 | **Ventajas**                                                                                                            | **Desventajas**                                                                                                             | **Casos de uso en sentimiento**                                                                                         | **Requisitos de datos/recursos**                                                       | **Preprocesamiento recomendado**                                                               | **Interpretabilidad**                                                  | **Escalabilidad**                                                   |
| --------------------------- | ----------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------- |
| **Naive Bayes**             | Probabilístico, lineal                    | Calcula la probabilidad de una clase (p. ej., positivo/negativo) asumiendo independencia condicional entre características (palabras).                                      | Muy rápido; robusto con datos escasos; buen baseline; fácil de entrenar y desplegar.                                    | Supuesto de independencia rara vez se cumple; sensibilidad a términos correlacionados; puede subestimar contexto.           | Clasificación rápida de tweets o reseñas; prototipado; sistemas con recursos limitados.                                 | Pocos datos para funcionar; bajo costo computacional; funciona bien con BOW/TF‑IDF.    | Limpieza básica, normalización, stopwords (opcional), n‑grams, TF‑IDF.                         | Alta (coeficientes y probabilidades son interpretables).               | Excelente en entrenamiento y predicción en grandes volúmenes.       |
| **SVM**                     | Margen máximo, lineal/no lineal (kernels) | Encuentra un hiperplano que maximiza el margen entre clases; kernels permiten fronteras no lineales.                                                                        | Alto rendimiento con datos de alta dimensión (texto); robusto al sobreajuste; efectivo con pocos ejemplos.              | Costoso con datasets muy grandes; tuning de C y kernel; menos eficiente en inferencia si hay muchos vectores soporte.       | Clasificación de polaridad y detección de toxicidad con TF‑IDF; competidor sólido en benchmarks clásicos.               | Más datos que NB; costo moderado-alto si se usan kernels complejos.                    | Tokenización, lematización, TF‑IDF, reducción de dimensión opcional.                           | Media (coeficientes en SVM lineal; kernels opacos).                    | Buena con SVM lineal; limitada con kernels en millones de muestras. |
| **Random Forest**           | Ensamble de árboles (bagging)             | Promedia múltiples árboles entrenados en subconjuntos de datos/características para reducir varianza.                                                                       | Maneja no linealidades; robusto a outliers; pocas suposiciones de distribución; estimación de importancia de variables. | Menos efectivo que lineales en texto esparso; modelos grandes; inferencia más lenta; menor rendimiento que SVM/BERT en NLP. | Cuando se combinan features de texto + metadata (hora, usuario, engagement); escenarios tabulares mixtos.               | Moderado en datos; más RAM/CPU que NB/SVM; tamaño del bosque afecta tiempo.            | Ingeniería de features (TF‑IDF) + variables contextuales; balanceo de clases.                  | Media (importancia de variables; árboles individuales interpretables). | Buena paralelización; tamaño del modelo crece con n\_árboles.       |
| **BERT (vía `reticulate`)** | Transformer preentrenado (deep learning)  | Usa atención bidireccional para capturar contexto; se **ajusta fino (fine-tuning)** para sentimiento. `reticulate` permite llamar modelos de Python (Hugging Face) desde R. | SOTA en NLP; entiende contexto, negaciones e ironía moderada; excelente generalización; embeddings reutilizables.       | Requiere GPU/TPU para entrenar; más datos y tiempo; complejidad de despliegue; menor interpretabilidad.                     | Clasificación de sentimiento multicategoría; detección de matices emocionales; análisis multilingüe; grandes volúmenes. | Alto: GPU recomendada, modelos preentrenados (p. ej., `bert-base-multilingual-cased`). | Tokenización WordPiece, truncado/padding, manejo de desbalance, limpieza mínima (no agresiva). | Baja (caja negra; se apoyan técnicas como LIME/SHAP/attention).        | Muy buena con inferencia batch en GPU; costo alto por instancia.    |




Ejemplo TF-IDF:

El TF‑IDF es una técnica que evalúa la importancia de cada palabra en un texto según su frecuencia local y su rareza global. En el análisis de sentimiento de una red social, resalta términos relevantes que influyen en la polaridad, mejorando la representación del texto y la precisión de los modelos de clasificación.

```r
tweets_dtm <- tweets_tokens %>%
  count(status_id, word) %>%
  bind_tf_idf(word, status_id, n)
```

---

Métricas de Evaluación

Si se usa ML supervisado:

* Accuracy
* Precision
* Recall
* F1-score
* Matriz de confusión
* AUC-ROC


Aquí tienes una **tabla detallada** que describe las métricas más comunes para evaluar modelos de análisis de sentimiento:

***

| **Métrica**             | **Qué es**                                                              | **Qué mide**                                        | **Cómo se interpreta**                                         | **Ventajas**                                                               | **Limitaciones**                                                         |
| ----------------------- | ----------------------------------------------------------------------- | --------------------------------------------------- | -------------------------------------------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| **Accuracy**            | Proporción de predicciones correctas sobre todas las predicciones.      | Rendimiento global del modelo.                      | Valores cercanos a 1 indican mejor desempeño general.          | Fácil de entender; útil cuando las clases están balanceadas.               | Engañosa si las clases están desbalanceadas.                             |
| **Precision**           | Proporción de verdaderos positivos entre todos los positivos predichos. | Calidad de las predicciones positivas.              | Alta precisión significa pocas falsas alarmas.                 | Ideal cuando el costo de un falso positivo es alto.                        | Puede ignorar muchos positivos reales si el recall es bajo.              |
| **Recall**              | Proporción de verdaderos positivos entre todos los positivos reales.    | Capacidad de detectar casos positivos.              | Alto recall significa que casi no se escapan positivos reales. | Útil cuando es importante no perder casos relevantes.                      | Puede aumentar a costa de más falsos positivos.                          |
| **F1-score**            | Media armónica entre precisión y recall.                                | Equilibrio entre precisión y recall.                | Alta puntuación indica balance adecuado entre ambas métricas.  | Ideal para clases desbalanceadas.                                          | No distingue la importancia de precisión o recall si una es más crítica. |
| **Matriz de confusión** | Tabla que muestra verdaderos/ falsos positivos y negativos.             | Comportamiento detallado del modelo en cada clase.  | Permite identificar errores específicos entre clases.          | Visualización clara del rendimiento.                                       | No resume el desempeño en un solo número.                                |
| **AUC-ROC**             | Área bajo la curva ROC (TPR vs. FPR).                                   | Capacidad del modelo para discriminar entre clases. | AUC cercano a 1 indica excelente separación entre clases.      | Robusta ante desbalance de clases; evalúa el modelo a diferentes umbrales. | Menos intuitiva; requiere probabilidades y no solo predicciones.         |

***


---

Consideraciones Avanzadas

Problemas técnicos

* Sarcasmo
* Ironía
* Ambigüedad semántica
* Bots
* Polarización artificial

Escalabilidad Big Data

En entornos productivos:

* Spark + sparklyr
* Bases NoSQL (MongoDB)
* Arquitectura Lambda
* Procesamiento streaming

_________________

Referencias 

> Saini, S., Punhani, R., Bathla, R., & Shukla, V. K. (2019). Sentiment analysis on Twitter data using R. ResearchGate. 
> Klinkhammer, D. (2022). Sentiment analysis with R: Natural language processing for semi-automated assessments of qualitative data (Working paper). 
> Schweinberger, M. (2022). Sentiment analysis in R. Tutorial basado en tidytext, sentimentr y lexicones emocionales. 


