# KNIME — Scraping de precio del azufre y extracción/clasificación de noticias

---

## 🧭 Objetivo

Construir un workflow **KNIME** que:

1. Obtenga y normalice **series de precios del azufre** (fuentes públicas/API o scraping permitido).
2. Ingesta y clasifique **noticias** relevantes (agro/fertilizantes) sobre “sulfur/sulfuro/azufre”.
3. Exporte datasets limpios (CSV/JSON) listos para **Power BI** / análisis.

---

## 🧱 Estructura del proyecto
| Componente | Descripción |
|-------------|-------------|
| `Reto Buconda.knwf` | Workflow de scraping de precios y noticias. |
| `Workflow knime.png` | Vista general del workflow. |

---

## ⚙️ Requisitos

* **KNIME Analytics Platform** ≥ 5.x

---

## 🔌 Fuentes de datos (ejemplos)

### Precios

* SunSirs: URL/API principal para precios (p.ej. proveedor público, índice spot, base propia).
* Formato recomendado de salida (**schema_prices**):

```json
{
  "date": "YYYY-MM-DD",
  "price_usd_t": 123.45,
  "source": "<string>",
  "frequency": "daily|weekly|monthly",
  "notes": "<string opcional>"
}
```

### Noticias (GDELT v2, ejemplo)

* Endpoint: `https://api.gdeltproject.org/api/v2/doc/doc?query=<QUERY>&mode=artlist&maxrecords=250&format=json`
* `QUERY` sugerida (agro/fertilizantes):

```
("sulfur" OR "sulphur" OR "azufre") AND (fertilizer OR fertiliser OR agriculture OR agro)
```

* Campos mínimos en salida (**schema_news**):

```json
{
  "date": "YYYY-MM-DD",
  "title": "<string>",
  "url": "<string>",
  "lang": "en|es|...",
  "source": "<domain>",
  "summary": "<string>",
  "tags": ["agriculture", "fertilizer"],
  "relevance": 0.0
}
```

> **Nota:** Respeta robots.txt y términos de uso de cualquier web. Prioriza APIs y feeds.

---

## 🛠️ Workflows KNIME 

### 1) `sulfur_prices.knwf`

**Bloques:**

1. **Web Interaction Start (Labs)** → inicializa la sesión de navegador (Selenium) para poder renderizar la página de SunSirs con JS.
2. **Navigator (Labs)** → navega a la URL objetivo del azufre en SunSirs y espera la carga completa del contenido.
3. **Content Retriever (Labs)** → captura el **HTML renderizado** (no sólo la fuente estática) para que el siguiente nodo pueda parsearlo.
4. **XPath** → extrae **fecha** y **precio** del HTML a columnas tabulares (`date_raw`, `price_raw`).
5. **String to Number** → convierte `price_raw` a tipo numérico (double), eliminando símbolos y separadores.
6. **Column Filter** → conserva únicamente las columnas necesarias (fecha, precio y/o fuente) y descarta ruido.
7. **String Manipulation** → limpia la fecha (trim, reemplazo de espacios no-break) y la normaliza a formato `yyyy-MM-dd` en una columna `date_clean`.
8. **String to Date&Time** → convierte `date_clean` a tipo **Local Date** (`date`).
9. **Column Resorter** → reordena columnas finales en el orden lógico: `date`, `price`, `source`.
10. **CSV Reader** → carga el **histórico previo** (`data/processed/sulfur_prices.csv`) para evitar duplicados cuando se añaden nuevas filas.
11. **String to Date&Time** (histórico) → asegura que la columna de fecha del histórico también sea **Local Date** y comparable.
12. **Reference Row Filter** → deja **solo los registros nuevos** comparando por `date` frente al histórico (evita duplicados).
13. **CSV Writer** → exporta/actualiza la serie consolidada en `/data/processed/sulfur_prices.csv` (append o overwrite según configuración).


### 2) `sulfur_news.knwf`

**Bloques:**

1. **Web Interaction Start (Labs)** → inicializa la sesión del navegador (Selenium) para habilitar la carga dinámica de los portales de noticias relacionados con el azufre.
2. **Navigator (Labs)** → abre la página de listados de artículos (por ejemplo, en SunSirs u otras fuentes de fertilizantes) y espera a que el contenido HTML se renderice completamente.
3. **Content Retriever (Labs)** → obtiene el código HTML completo del listado de noticias para poder analizarlo mediante expresiones XPath.
4. **XPath** → extrae los **títulos**, **enlaces (URLs)** y **fechas** de publicación de los artículos.
5. **String Manipulation** → elimina espacios en blanco, saltos de línea o caracteres no imprimibles en los textos.
6. **String Manipulation (2)** → normaliza el formato de las fechas (`yyyy-MM-dd`) y asegura que todas las URLs comiencen con `https://`.
7. **Web Interaction Start (Labs)** (segunda rama) → inicia una nueva sesión Selenium para navegar a las páginas individuales de las noticias.
8. **Navigator (Labs)** (2) → recorre cada URL obtenida para capturar el texto completo de las noticias y los posibles resúmenes.
9. **Content Retriever (Labs)** (2) → descarga el HTML renderizado de cada noticia individual.
10. **XPath** (2) → extrae el cuerpo principal del texto o el resumen de cada noticia.
11. **XPath** (3)** → recoge información adicional (fuente, categoría o etiquetas del artículo).
12. **Duplicate Row Filter** → elimina duplicados basándose en `url` o la combinación `title + date`.
13. **Row Filter** → mantiene únicamente las noticias relevantes sobre azufre en el contexto **agrícola/fertilizante**, filtrando por palabras clave (`sulfur`, `azufre`, `fertilizer`, `agriculture`).
14. **Column Filter** → conserva solo las columnas finales necesarias (`date`, `title`, `url`, `source`, `summary`).
15. **Concatenate** → une los datos procedentes de las dos ramas (listado y detalle) en una tabla final consolidada.
16. **CSV Writer** → exporta el conjunto de noticias limpio y deduplicado a `/data/processed/sulfur_news.csv` (o `.json` si se desea en formato JSON).


**Tips útiles**

* Para **espacios en blanco** tras `XPath/JSON`: usa **String Manipulation** con `strip($col$)` o **Column Expressions**: `replaceChars(column("title"), "\u00A0", " ")` y `strip()`.
* Si una página devuelve **missing**, valida cabeceras con **GET Request** (User-Agent) y maneja códigos 429/403 con **Wait...** + **Retry**.
* Logs: añade **Table Writer** a `/data/interim/log_runs.csv` con marca temporal.


### 3) Python Script vaderSentiment (opcional)

```
import pandas as pd
import numpy as np
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer

df = input_table_1.copy()

analyzer = SentimentIntensityAnalyzer()

def score_text(t):
    if not isinstance(t, str) or not t.strip():
        return {"neg": np.nan, "neu": np.nan, "pos": np.nan, "compound": np.nan}
    return analyzer.polarity_scores(t)

scores = df["body"].apply(score_text).apply(pd.Series)
df["vader_neg"] = scores["neg"]
df["vader_neu"] = scores["neu"]
df["vader_pos"] = scores["pos"]
df["vader_compound"] = scores["compound"]

def label(c):
    if pd.isna(c): return None
    if c >= 0.05:  return "positive"
    if c <= -0.05: return "negative"
    return "neutral"

df["sentiment_label"] = df["vader_compound"].apply(label)

output_table_1 = df
```

---

