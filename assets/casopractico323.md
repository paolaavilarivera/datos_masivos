#### Caso de Uso: Análisis de Sentimiento para evaluar la percepción social de una política pública

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

```r
tweets_tokens <- tweets_clean %>%
  unnest_tokens(word, text)
```

Eliminar stopwords en español:

```r
data("stop_words")

stop_words_es <- stop_words %>%
  filter(lexicon == "snowball")

tweets_tokens <- tweets_tokens %>%
  anti_join(stop_words_es, by = "word")
```

---

Análisis de Sentimiento Lexicográfico

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

```r
sentiment_values <- get_sentiment(tweets_clean$text, method = "syuzhet")

tweets_clean$sentiment <- sentiment_values

mean(sentiment_values)
```

---

Extensión: Machine Learning Supervisado

Para un enfoque más avanzado:

1. Crear dataset etiquetado manualmente
2. Vectorización TF-IDF
3. Modelo:

   * Naive Bayes
   * SVM
   * Random Forest
   * BERT (vía `reticulate`)

Ejemplo TF-IDF:

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
