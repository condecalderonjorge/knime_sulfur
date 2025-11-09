# KNIME — Scraping de precio del azufre y extracción/clasificación de noticias

---

## 🧭 Objetivo

Construir un workflow **KNIME** que:

1. Obtenga y normalice **series de precios del azufre** (fuentes públicas/API o scraping permitido).
2. Ingesta y clasifique **noticias** relevantes (agro/fertilizantes) sobre “sulfur/sulfuro/azufre”.
3. Exporte datasets limpios (CSV/JSON) listos para **Power BI** / análisis.

---

## 📁 Estructura del repo

```
knime-sulfur-news/
├─ README.md                     # Este archivo
├─ LICENSE                       # MIT por defecto (opcional)
├─ .gitignore
├─ .gitattributes                # (opcional) Git LFS para ficheros grandes
├─ /workflows/
│   ├─ sulfur_prices.knwf        # Workflow KNIME precios 
│   └─ sulfur_news.knwf          # Workflow KNIME noticias 
├─ /data/
│   ├─ raw/                      # Descargas originales (no versionar si son pesadas)
│   ├─ interim/                  # Limpiezas intermedias
│   └─ processed/                # Salidas finales (CSV/Parquet/JSON)
├─ /docs/
│   ├─ schema_prices.json        # Esquema de salida precios
│   ├─ schema_news.json          # Esquema de salida noticias
│   └─ screenshots/              # Capturas de workflows / ejemplos
└─ /notebooks/                   # (opcional) Validaciones en Python/R
```

---

## ⚙️ Requisitos

* **KNIME Analytics Platform** ≥ 5.x

---

## 🔌 Fuentes de datos (ejemplos)

### Precios

* `TODO:` URL/API principal para precios (p.ej. proveedor público, índice spot, base propia).
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

1. **HTTP Retriever / CSV Reader / Excel Reader** → ingesta desde API/archivo.
2. **String/Column Manipulation** → limpieza (trim, lower, parseo numérico).
3. **Date&Time** → normalización de fecha a `YYYY-MM-DD` y *timezone*.
4. **Missing Value** → imputación/descartes.
5. **Rule Engine** → filtrar anómalos (negativos, outliers evidentes).
6. **GroupBy / Window** → agregaciones (W/M) si procede.
7. **Column Rename** → conformar esquema final.
8. **CSV/Parquet Writer** → `/data/processed/sulfur_prices.csv`.

### 2) `sulfur_news.knwf`

**Bloques:**

1. **GET Request** → GDELT (`mode=artlist`, `format=json`, `maxrecords` controlado).
2. **JSON to Table** / **JSON Path** → expandir array `articles`.
3. **String Manipulation (Multi Column)** → `strip()` espacios, normalizar idioma.
4. **Regex Extractor** → dominio de `url` → `source`.
5. **Row Filter** → incluir solo agro/fertilizantes (consulta o palabras clave secundarias).
6. **Duplicate Row Filter** → por `url` o `title+date`.
7. **Rule Engine** → *scoring* sencillo de relevancia por términos (p.ej. +1 “fertilizer”, +1 “agriculture”, -1 “battery”).
8. **CSV/JSON Writer** → `/data/processed/sulfur_news.json` y `.csv`.

**Tips útiles**

* Para **espacios en blanco** tras `XPath/JSON`: usa **String Manipulation** con `strip($col$)` o **Column Expressions**: `replaceChars(column("title"), "\u00A0", " ")` y `strip()`.
* Si una página devuelve **missing**, valida cabeceras con **GET Request** (User-Agent) y maneja códigos 429/403 con **Wait...** + **Retry**.
* Logs: añade **Table Writer** a `/data/interim/log_runs.csv` con marca temporal.

---

