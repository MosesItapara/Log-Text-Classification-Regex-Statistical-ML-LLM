# 🪵 Hybrid Log (Text) Classification System

A hybrid log classification system that combines **Regular Expressions**, **Sentence Transformers + Logistic Regression** and **Large Language Models (LLMs)** to accurately classify log messages across varying levels of complexity and data availability.

---

## 📌 Table of Contents

- [Overview](#overview)
- [Classification Approaches](#classification-approaches)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
- [Running the Server](#running-the-server)
- [API Usage](#api-usage)
- [Input Format](#input-format)
- [Output Format](#output-format)
- [Training Your Own Models](#training-your-own-models)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)

---

## Overview

Log classification is a critical step in monitoring, debugging, and maintaining software systems. This project implements a **three-tier hybrid classification pipeline** that intelligently routes log messages to the most appropriate classification method based on pattern complexity and data availability.

The system is exposed via a **FastAPI** REST API, allowing seamless integration with existing log pipelines or monitoring tools through a simple CSV upload interface.

---

## Classification Approaches

### 1. 🔤 Regular Expressions (Regex)
- Targets **simple, predictable and well-defined** log patterns.
- Fastest classification method — zero model inference overhead.
- Rules are defined manually and can be extended in `training/`.
- Best suited for structured patterns like error codes, HTTP status codes or fixed-format system messages.

### 2. 🤖 Sentence Transformer + Logistic Regression
- Handles **complex patterns where sufficient labeled training data exists**.
- Log messages are converted into semantic embeddings using a [Sentence Transformer](https://www.sbert.net/) model.
- A **Logistic Regression** classifier is trained on top of these embeddings.
- Strikes a balance between accuracy and inference speed.
- Pre-trained models are saved in the `models/` directory and loaded at runtime.

### 3. 🧠 LLM (Large Language Model via Groq)
- Acts as a **fallback** for complex or ambiguous patterns where labeled training data is scarce or unavailable.
- Leverages Groq-hosted LLMs (e.g., `llama-3.3-70b-versatile`) for zero-shot or few-shot classification.
- Ensures the system remains robust even when the other two methods cannot produce a confident classification.

---

## Project Structure

```
TEXTCLASSIFICATION/
│
├── training/                   # Training scripts and regex rules
│   ├── train_model.py          # Train Sentence Transformer + Logistic Regression
│   └── regex_patterns.py       # Predefined regex classification rules
│
├── models/                     # Saved model artifacts
│   ├── logistic_model.pkl      # Trained Logistic Regression model
│   └── label_encoder.pkl       # Label encoder for target classes
│
├── resources/                  # Sample data, outputs, and assets
│   ├── test.csv                # Sample input CSV for testing
│   └── output.csv              # Sample output after classification
│
├── classify.py                 # Core classification logic and CSV pipeline
├── processor_llm.py            # LLM-based classification via Groq API
├── server.py                   # FastAPI server (main entry point)
├── requirements.txt            # Python dependencies
└── README.md
```

---

## Prerequisites

- Python **3.8+**
- A [Groq API key](https://console.groq.com/) (for LLM-based classification)
- `pip` for installing dependencies

---

## Setup Instructions

### 1. Clone the Repository

```bash
git clone https://github.com/MosesItapara/textclassification.git
cd textclassification
```

### 2. Create and Activate a Virtual Environment

```bash
# Create virtual environment
python -m venv venv

# Activate — Windows
venv\Scripts\activate

# Activate — macOS/Linux
source venv/bin/activate
```

### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure Your Groq API Key

Create a `.env` file in the root directory and add your Groq API key:

```env
GROQ_API_KEY=your_groq_api_key_here
```

> ⚠️ Never commit your `.env` file to version control. Add it to `.gitignore`.

---

## Running the Server

Start the FastAPI server with:

```bash
uvicorn server:app --reload
```

The server will be available at:

| Endpoint | Description |
|----------|-------------|
| `http://127.0.0.1:8000/` | Main classification endpoint |
| `http://127.0.0.1:8000/docs` | Interactive Swagger UI documentation |
| `http://127.0.0.1:8000/redoc` | Alternative ReDoc API documentation |

---

## API Usage

### `POST /classify`

Upload a CSV file containing log messages to receive a classified output CSV.

**Request**

```bash
curl -X POST "http://127.0.0.1:8000/classify" \
  -H "accept: application/json" \
  -F "file=@resources/test.csv"
```

**Response**

Returns a CSV file with an additional `target_label` column containing the classification result for each log entry.

---

## Input Format

The uploaded CSV file must contain the following columns:

| Column | Type | Description |
|--------|------|-------------|
| `source` | `string` | The origin system or service that produced the log |
| `log_message` | `string` | The raw log message to be classified |

**Example (`test.csv`)**

```csv
source,log_message
auth-service,User login failed: invalid credentials for user admin
payment-service,Transaction completed successfully. Amount: $250.00
web-server,GET /api/v1/users 200 OK 45ms
db-service,Connection timeout after 30s retrying...
```

---

## Output Format

The output CSV includes all original columns plus:

| Column | Type | Description |
|--------|------|-------------|
| `target_label` | `string` | The predicted classification label for the log entry |

**Example (`output.csv`)**

```csv
source,log_message,target_label
auth-service,User login failed: invalid credentials for user admin,Authentication Failure
payment-service,Transaction completed successfully. Amount: $250.00,Transaction Success
web-server,GET /api/v1/users 200 OK 45ms,HTTP Request
db-service,Connection timeout after 30s retrying...,Database Error
```

---

## Training Your Own Models

To retrain the Sentence Transformer + Logistic Regression model on your own labeled data:

1. Prepare a CSV with `log_message` and `label` columns.
2. Place it in the `resources/` directory.
3. Run the training script:

```bash
python training/train_model.py
```

Trained model artifacts will be saved to the `models/` directory and automatically picked up by the classification pipeline.

---

## Configuration

| Setting | Location | Description |
|---------|----------|-------------|
| Groq model name | `processor_llm.py` | Change the LLM model (e.g., `llama-3.3-70b-versatile`) |
| Regex patterns | `training/regex_patterns.py` | Add or modify regex classification rules |
| Confidence threshold | `classify.py` | Tune when to fall back from ML model to LLM |

**Recommended Groq models** (as of 2026):

| Model | Best For |
|-------|----------|
| `llama-3.3-70b-versatile` | General-purpose classification (recommended) |
| `qwen-qwq-32b` | Complex reasoning and ambiguous patterns |
| `deepseek-r1-distill-qwen-32b` | Reasoning-heavy log analysis |

---

## Troubleshooting

**`OSError: Invalid argument` when reading CSV**
> Use forward slashes or raw strings for file paths on Windows:
> ```python
> classify_csv("resources/test.csv")  # ✅
> classify_csv(r"resources\test.csv") # ✅
> classify_csv("resources\test.csv")  # ❌ \t is a tab character
> ```

**`groq.BadRequestError: model decommissioned`**
> The model specified in `processor_llm.py` is no longer supported. Update it to a current model such as `llama-3.3-70b-versatile`. See [Groq deprecation docs](https://console.groq.com/docs/deprecations).

**`ModuleNotFoundError`**
> Ensure your virtual environment is activated and dependencies are installed:
> ```bash
> venv\Scripts\activate   # Windows
> pip install -r requirements.txt
> ```

---

## License

This project is licensed under the [MIT License](LICENSE).

---

## Acknowledgements

- [Groq](https://groq.com/) for ultra-fast LLM inference
- [Sentence Transformers](https://www.sbert.net/) for semantic embeddings
- [FastAPI](https://fastapi.tiangolo.com/) for the REST API framework
