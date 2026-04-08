# 🌾 Mandi Mitra — AI Agricultural Market Advisor

> Mandi Mitra is a multilingual AI chatbot that helps Indian farmers find the **best mandi (market)** to sell their crop at the highest price. Built on Databricks using Llama 4 Maverick for natural language understanding — supporting 8 Indian languages.

---

## Architecture Diagram

![Upaj.ai Architecture](architecture.png)

> **How Databricks components connect:** The farmer's query enters the Gradio web UI → Llama 4 Maverick (via Databricks Model Serving + Databricks SDK) extracts intent and translates district names → pre-aggregated mandi price data (`mandi_data.json`, exported from the Gold Delta Lake table) is queried locally → net return is ranked → Llama 4 generates a reply in the farmer's native language.

---

## What it Does

Mandi Mitra takes a farmer's question in their native language (e.g., *"मला पुण्यात कांदा विकायचा आहे"*), identifies the crop and district using Llama 4 Maverick, looks up pre-aggregated mandi prices, and replies in the farmer's language with the best mandi name, price per quintal, and estimated profit.

**Supported crops:** Tomato · Onion · Potato · Wheat

**Supported languages:** English · Hinglish · Hindi (हिन्दी) · Tamil (தமிழ்) · Telugu (తెలుగు) · Marathi (मराठी) · Bengali (বাংলা) · Punjabi (ਪੰਜਾਬੀ)

---

## Databricks Technologies Used

| Technology | How it's used |
|---|---|
| **Databricks Model Serving** | Hosts `databricks-llama-4-maverick` as a live LLM endpoint |
| **Databricks SDK** (`WorkspaceClient`) | Calls the LLM endpoint from Python |
| **Delta Lake (Gold table)** | Source of mandi price data (exported to `mandi_data.json`) |
| **Apache Spark + Spark MLlib** | Used to train Random Forest price model and generate the Gold table |
| **MLflow** | Experiment tracking — logs RMSE, MAE, params, model artifact |

**Open-source model:** `meta-llama/llama-4-maverick` via `databricks-llama-4-maverick`

---

## How to Run

### Prerequisites

- Python 3.10+
- Databricks workspace with access to `databricks-llama-4-maverick` model serving endpoint
- A Databricks personal access token

### Step 1 — Clone the repo

```bash
git clone https://github.com/arpitjainiitd/Bharat-Bricks-2026-Hackathon.git
cd Bharat-Bricks-2026-Hackathon
```

### Step 2 — Install dependencies

```bash
pip install gradio==4.44.0 databricks-sdk==0.20.0 pydantic==2.8.2 requests==2.31.0 pandas==2.0.3
```

### Step 3 — Set environment variables

```bash
export DATABRICKS_HOST=https://your-workspace.azuredatabricks.net
export DATABRICKS_TOKEN=your-personal-access-token
export SARVAM_API_KEY=your-sarvam-key
```

Or create a `.env` file:

```
DATABRICKS_HOST=https://your-workspace.azuredatabricks.net
DATABRICKS_TOKEN=dapi_xxxxxxxxxxxx
SARVAM_API_KEY=sk_xxxxxxxxxxxx
```

### Step 4 — Add mandi data

Ensure `mandi_data.json` is present in the same folder as `app.py`. To regenerate it from Databricks, run this in a Databricks notebook:

```python
df = spark.table("default.gold_mandi_features")
pdf = df.select("commodity", "district_name", "market_center", "modal_price") \
        .groupBy("commodity", "district_name", "market_center") \
        .agg({"modal_price": "avg"}) \
        .withColumnRenamed("avg(modal_price)", "avg_price") \
        .toPandas()

import json
with open("mandi_data.json", "w") as f:
    json.dump(pdf.to_dict(orient="records"), f)
```

Download the file and place it in the repo root.

### Step 5 — Run the app

```bash
python app.py
```

The app launches at `http://localhost:8000`

---

## File Structure

```
Bharat-Bricks-2026-Hackathon/
├── app.py               ← Gradio chatbot app
├── mandi_data.json      ← Pre-aggregated mandi price data
├── requirements.txt     ← Python dependencies
├── architecture.png     ← Architecture diagram
└── README.md
```

---

## Demo Steps

Follow these exact steps to reproduce the demo:

**1.** Run `python app.py` and open `http://localhost:8000`

**2.** Select language → choose **`हिन्दी`** from the dropdown

**3.** Fill in the inputs:
- District: `pune`
- Quintals: `10`

**4.** Type this prompt and click **मंडी मित्र से पूछें ➤**
```
पुण्यात टोमॅटो कुठे विकावा?
```

**5.** Observe the Hindi reply with mandi name, price per quintal, and profit estimate

**6.** Switch language to **`English`** and type:
```
Where should I sell onion in Nashik?
```

**7.** Observe the reply switches to English automatically

### Expected output
```
🤖 Nira mandi in Pune offers the best price for tomatoes at ₹1,842/quintal.
   For 10 quintals, your estimated profit is ₹18,170. Jai Kisan!
```

---

## Quantitative Accuracy Metrics (Bonus)

The mandi price data is sourced from a Random Forest model trained on `default.gold_mandi_features` using Spark MLlib. Metrics logged to MLflow:

| Metric | Description |
|---|---|
| **RMSE** | Root Mean Squared Error on held-out test set |
| **MAE** | Mean Absolute Error — average ₹ error per quintal |

Model config: `numTrees=10`, `maxDepth=5`, `maxBins=2000`

Features: `commodity`, `district_name`, `market_center`, `month`, `day_of_week`, `avg_temp_c`, `avg_rainfall_mm`, `is_harvest_season`

---

## MLflow Experiment Logs (Bonus)

Training run logged as `MandiMitra_RF_Forecaster` in Databricks MLflow:

```python
mlflow.log_metric("rmse", rmse)
mlflow.log_metric("mae",  mae)
mlflow.log_param("numTrees", 10)
mlflow.log_param("maxDepth", 5)
```

---

## BhashaBench Evaluation (Bonus)

Fixed crop+district queries sent in each language. Pass criteria: correct script, valid mandi name, ₹ price present.

| Language | Script | Result |
|---|---|---|
| Hindi | Devanagari | ✅ Pass |
| Marathi | Devanagari | ✅ Pass |
| Bengali | Bengali | ✅ Pass |
| Tamil | Tamil | ✅ Pass |
| Telugu | Telugu | ✅ Pass |
| Punjabi | Gurmukhi | ✅ Pass |
| Hinglish | Latin | ✅ Pass |
| English | Latin | ✅ Pass |

---

## Team

Built at **Bharat Bricks Hacks 2026 — IIT Delhi**
