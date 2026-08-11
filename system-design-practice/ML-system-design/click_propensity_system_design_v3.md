# Click Propensity Model — End-to-End System Design

**Version:** 3.0  
**Scope:** Production-grade architecture, weekend-deployable via Docker Compose  
**Dataset:** Avazu CTR Prediction (Kaggle)  
**Stack:** LightGBM/ONNX · Triton · FastAPI · Redis · Kafka · React  
**Cost to run:** $0 — everything containerised on localhost

---

## 1. Guiding Principles

- **Real stack, real components.** Redis, Kafka, and Triton are not replaced by toys. They run as Docker containers locally, identical APIs to production.
- **Docker Compose is the deployment unit.** One `docker-compose up` brings the entire system online.
- **ONNX as the model format.** Train in LightGBM, export to ONNX, serve via Triton — this is the standard production path.
- **Kafka single-broker, no replication.** Same API as a multi-broker cluster, zero ops overhead on a laptop.
- **Triton on CPU.** Same gRPC/HTTP serving API as GPU Triton, just slower — fine for demo throughput.
- **React frontend.** A real SPA, not a Streamlit prototype.
- **Avazu as the dataset.** Real impression + click log with genuine user proxy (`device_id`), item proxy (`app_id`), timestamp (`hour`), and position (`banner_pos`) — no synthetic workarounds needed.

---

## 2. Dataset: Avazu CTR Prediction

**Source:** `https://www.kaggle.com/c/avazu-ctr-prediction`  
**Pre-sampled 50K version (prototyping):** `https://www.kaggle.com/datasets/gauravduttakiit/avazu-ctr-prediction-with-random-50k-rows`

### 2.1 Raw Schema

| Column | Type | Description |
|---|---|---|
| `id` | string | Impression identifier |
| `click` | int (0/1) | **Target label** |
| `hour` | int | Timestamp — format YYMMDDHH (e.g. `14091123` = 23:00 Sept 11 2014) |
| `C1` | categorical | Anonymized user signal |
| `banner_pos` | int | Ad banner position on page — maps to `position` |
| `site_id` | categorical | Site identifier |
| `site_domain` | categorical | Site domain |
| `site_category` | categorical | Site content category |
| `app_id` | categorical | App identifier — maps to `item_id` |
| `app_domain` | categorical | App domain |
| `app_category` | categorical | App content category — maps to `category` |
| `device_id` | categorical | Device identifier — maps to `user_id` |
| `device_ip` | categorical | Device IP (too high cardinality — dropped) |
| `device_model` | categorical | Device model |
| `device_type` | int | Device type (phone/tablet/desktop) |
| `device_conn_type` | int | Connection type (WiFi/3G/LTE etc.) |
| `C14–C21` | categorical | Anonymized ad/context features |

### 2.2 Download

```bash
# Full dataset (~6GB, 40M rows) — recommended for training
kaggle competitions download -c avazu-ctr-prediction

# Pre-sampled 50K rows — use to get pipeline running before full training
kaggle datasets download -d gauravduttakiit/avazu-ctr-prediction-with-random-50k-rows
```

### 2.3 Sampling Strategy for Weekend Use

The full dataset is 40M rows. For the weekend build, use a time-ordered 2M row sample:

```python
import pandas as pd

df = pd.read_csv("data/train.gz", compression="gzip")
df = df.sort_values("hour")          # ensure chronological order
df_sample = df.head(2_000_000)       # first 2M rows (earliest days)
df_sample.to_csv("data/avazu_2m.csv", index=False)
```

This preserves temporal ordering, which is required for the time-based train/val split.

---

## 3. Architecture Overview

```
╔══════════════════════════════════════════════════════════════════════╗
║                         OFFLINE PIPELINE                             ║
║                                                                      ║
║  avazu_2m.csv     Feature Eng.       Training          Export        ║
║  (Avazu sample)──►(avazu_features──► (LightGBM)  ──►  (ONNX)        ║
║                    .py)                                model.onnx    ║
║                         │                                   │        ║
║                         ▼                                   ▼        ║
║                   Redis (seed device_id          Triton Model        ║
║                   + app_id features)             Repository          ║
╚══════════════════════════════════════════════════════════════════════╝
                                          │
                                          │ model.onnx pushed to
                                          │ /models/propensity/
                                          ▼
╔══════════════════════════════════════════════════════════════════════╗
║                         ONLINE SYSTEM                                ║
║                                                                      ║
║  ┌──────────────┐   REST    ┌──────────────────────────────────────┐ ║
║  │              │ ────────► │         Backend API (FastAPI)         │ ║
║  │  React SPA   │           │                                       │ ║
║  │  (Vite)      │ ◄──────── │  /recommend   /impression   /click   │ ║
║  │              │   JSON    │                                       │ ║
║  │  • Device    │           │         │              │              │ ║
║  │    selector  │           │         ▼              ▼              │ ║
║  │  • App grid  │           │  ┌─────────────┐ ┌──────────────┐    │ ║
║  │    (ads)     │           │  │Feature      │ │Triton        │    │ ║
║  │  • p(click)  │           │  │Assembly     │ │Inference     │    │ ║
║  │    badges    │           │  │             │ │Server        │    │ ║
║  │  • Click     │           │  │ Redis GET   │ │              │    │ ║
║  │    logging   │           │  │ device:{id} │ │ HTTP /v2/    │    │ ║
║  │              │           │  │ app:{id}    │ │ models/      │    │ ║
║  └──────────────┘           │  └──────┬──────┘ │ propensity/  │    │ ║
║                             │         │        │ infer        │    │ ║
║                             │         └───────►│              │    │ ║
║                             │                  └──────┬───────┘    │ ║
║                             │                         │ scores     │ ║
║                             │                         ▼            │ ║
║                             │              ranked apps + p(click)  │ ║
║                             └──────────────────────────────────────┘ ║
║                                          │                           ║
║                             impression / click events                ║
║                                          │                           ║
║                                          ▼                           ║
║                             ┌────────────────────────┐              ║
║                             │   Kafka (single broker) │              ║
║                             │                         │              ║
║                             │   topic: impressions    │              ║
║                             │   topic: clicks         │              ║
║                             └────────────┬────────────┘              ║
║                                          │                           ║
║                                          ▼                           ║
║                             ┌────────────────────────┐              ║
║                             │   Event Consumer        │              ║
║                             │   (Python worker)       │              ║
║                             │                         │              ║
║                             │   • Updates Redis CTR   │              ║
║                             │     (EMA sliding window)│              ║
║                             │   • Writes to Postgres  │              ║
║                             └────────────────────────┘              ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 4. Component Inventory

| Component | Technology | Role | Weekend Shortcut |
|---|---|---|---|
| Frontend | React + Vite | UI rendering, event firing | None — build it properly |
| Backend API | FastAPI + Uvicorn | Orchestration, feature assembly, Kafka producer | None |
| Feature Store | Redis 7 (standalone) | Sub-millisecond device + app feature reads | No cluster/sentinel |
| Model Server | Triton Inference Server (CPU) | ONNX model serving via HTTP | CPU image, no GPU |
| Event Bus | Kafka (Confluent single-broker) | Impression + click event streaming | Single partition, no replication |
| Event Consumer | Python (confluent-kafka) | Consume events, update Redis CTR | None |
| Storage | PostgreSQL (optional) | Durable event log | Can skip Day 1, add Day 2 |
| Model Format | ONNX (exported from LightGBM) | Triton-compatible inference | None |
| Orchestration | Docker Compose | Run all services locally | Replaces Kubernetes |
| Training | Python, LightGBM, pandas | Offline model training on Avazu sample | None |

---

## 5. Project Layout

```
click-propensity/
│
├── docker-compose.yml
│
├── data/
│   ├── train.gz                     # Raw Avazu download (from Kaggle)
│   └── avazu_2m.csv                 # Sampled 2M rows (generated by prep.py)
│
├── offline/
│   ├── prep.py                      # Sample + sort Avazu data
│   ├── avazu_features.py            # Feature engineering (shared by train + seed)
│   ├── train.py                     # Train LightGBM, export to ONNX
│   ├── seed_redis.py                # Compute per-device/per-app stats -> Redis
│   └── requirements.txt
│
├── encoders/
│   ├── label_encoders.pkl           # LabelEncoders fitted on training set
│   ├── device_ctr.pkl               # Per-device historical CTR dict
│   ├── app_ctr.pkl                  # Per-app historical CTR dict
│   └── calibrator.pkl               # Platt scaler
│
├── api/
│   ├── main.py                      # FastAPI app
│   ├── features.py                  # Redis feature assembly (Avazu schema)
│   ├── triton_client.py             # Triton HTTP client wrapper
│   ├── kafka_producer.py            # Impression + click producers
│   ├── schemas.py                   # Pydantic request/response models
│   ├── Dockerfile
│   └── requirements.txt
│
├── consumer/
│   ├── worker.py                    # Kafka consumer, Redis CTR updater
│   ├── Dockerfile
│   └── requirements.txt
│
├── triton/
│   └── models/
│       └── propensity/
│           ├── config.pbtxt         # Triton model config (13 input features)
│           └── 1/
│               └── model.onnx       # Generated by train.py
│
└── frontend/
    ├── package.json
    ├── vite.config.js
    └── src/
        ├── main.jsx
        ├── App.jsx
        ├── components/
        │   ├── DeviceSelector.jsx   # Pick a device_id (proxy for user)
        │   ├── AdGrid.jsx           # Ranked ad cards
        │   ├── AdCard.jsx           # App ad with p(click) badge
        │   └── ScoreBadge.jsx
        └── api/
            └── client.js
```

---

## 6. Feature Engineering

### 6.1 Avazu Column to Model Feature Mapping

| Model Feature | Avazu Source | Type | Notes |
|---|---|---|---|
| `device_hist_ctr` | computed from `device_id` + `click` | float | Per-device historical CTR from training set |
| `device_recent_ctr` | Redis EMA updated by consumer | float | Sliding window CTR, init = hist_ctr |
| `app_hist_ctr` | computed from `app_id` + `click` | float | Per-app historical CTR from training set |
| `app_recent_ctr` | Redis EMA updated by consumer | float | Sliding window CTR, init = hist_ctr |
| `banner_pos` | `banner_pos` | int | Position of ad on page (0–7) |
| `site_category` | `site_category` (label-encoded) | int | Content category of host site |
| `app_category` | `app_category` (label-encoded) | int | Category of advertised app |
| `device_type` | `device_type` | int | 0=phone, 1=tablet, 4=desktop |
| `device_conn_type` | `device_conn_type` | int | 0=unknown, 2=WiFi, 3=cellular |
| `hour_of_day` | extracted from `hour` field | int | 0–23 |
| `day_of_week` | extracted from `hour` field | int | 0=Mon, 6=Sun |
| `C1` | `C1` (label-encoded) | int | Anonymized user signal |
| `device_x_app_cat` | `device_type` x `app_category` | int | Feature cross |

**Total: 13 features.** Triton config must set `dims: [13]`.

Dropped columns: `device_ip` (too high cardinality), `device_model`, `site_id`, `site_domain`, `app_domain`, `id`, `C14-C21` (can be added later as additional categorical embeddings).

### 6.2 Shared Feature Module (`offline/avazu_features.py`)

This module is imported by both `train.py` and `seed_redis.py` to guarantee identical transformations — the single most important file for preventing training-serving skew.

```python
import pandas as pd
import numpy as np
import pickle
from sklearn.preprocessing import LabelEncoder
from datetime import datetime

FEATURE_COLS = [
    "device_hist_ctr", "device_recent_ctr",
    "app_hist_ctr",    "app_recent_ctr",
    "banner_pos",      "site_category",    "app_category",
    "device_type",     "device_conn_type",
    "hour_of_day",     "day_of_week",
    "C1",              "device_x_app_cat",
]
TARGET   = "click"
CAT_COLS = ["site_category", "app_category", "C1"]

def parse_hour(hour_int):
    s = str(int(hour_int))
    hour_of_day = int(s[6:8])
    dt = datetime.strptime(s[:6], "%y%m%d")
    return hour_of_day, dt.weekday()

def fit_encoders(df):
    encoders = {}
    for col in CAT_COLS:
        le = LabelEncoder()
        le.fit(df[col].astype(str).fillna("__missing__"))
        encoders[col] = le
    pickle.dump(encoders, open("encoders/label_encoders.pkl", "wb"))
    return encoders

def load_encoders():
    return pickle.load(open("encoders/label_encoders.pkl", "rb"))

def apply_encoders(df, encoders):
    df = df.copy()
    for col, le in encoders.items():
        known = set(le.classes_)
        df[col] = df[col].astype(str).fillna("__missing__")
        df[col] = df[col].apply(lambda x: x if x in known else "__missing__")
        df[col] = le.transform(df[col])
    return df

def compute_ctr_stats(df):
    device_ctr = df.groupby("device_id")["click"].mean().to_dict()
    app_ctr    = df.groupby("app_id")["click"].mean().to_dict()
    return device_ctr, app_ctr

def engineer_features(df, encoders, device_ctr, app_ctr):
    df = df.copy()
    parsed = df["hour"].apply(parse_hour)
    df["hour_of_day"] = parsed.apply(lambda x: x[0])
    df["day_of_week"]  = parsed.apply(lambda x: x[1])

    global_device_ctr = float(np.mean(list(device_ctr.values())))
    global_app_ctr    = float(np.mean(list(app_ctr.values())))
    df["device_hist_ctr"]   = df["device_id"].map(device_ctr).fillna(global_device_ctr)
    df["app_hist_ctr"]      = df["app_id"].map(app_ctr).fillna(global_app_ctr)
    df["device_recent_ctr"] = df["device_hist_ctr"]
    df["app_recent_ctr"]    = df["app_hist_ctr"]

    df = apply_encoders(df, encoders)
    df["device_x_app_cat"] = df["device_type"] * 100 + df["app_category"]

    return df[FEATURE_COLS + [TARGET]]
```

---

## 7. Offline Training (`offline/train.py`)

```python
import pandas as pd
import numpy as np
import lightgbm as lgb
import pickle, os
from sklearn.calibration import CalibratedClassifierCV
from sklearn.metrics import roc_auc_score, log_loss
from avazu_features import (fit_encoders, engineer_features,
                             compute_ctr_stats, FEATURE_COLS, TARGET)

os.makedirs("encoders",                       exist_ok=True)
os.makedirs("triton/models/propensity/1",     exist_ok=True)

print("Loading Avazu sample...")
df = pd.read_csv("data/avazu_2m.csv",
                 dtype={"device_id": str, "app_id": str, "site_category": str,
                        "app_category": str, "C1": str})
df = df.sort_values("hour").reset_index(drop=True)

split    = int(len(df) * 0.8)
train_df = df.iloc[:split].copy()
val_df   = df.iloc[split:].copy()
print(f"Train: {len(train_df):,} | Val: {len(val_df):,} | "
      f"Train CTR: {train_df['click'].mean():.4f}")

# Fit encoders and CTR stats on training set ONLY
encoders = fit_encoders(train_df)
device_ctr, app_ctr = compute_ctr_stats(train_df)
pickle.dump(device_ctr, open("encoders/device_ctr.pkl", "wb"))
pickle.dump(app_ctr,    open("encoders/app_ctr.pkl",    "wb"))

train_feat = engineer_features(train_df, encoders, device_ctr, app_ctr)
val_feat   = engineer_features(val_df,   encoders, device_ctr, app_ctr)

X_train, y_train = train_feat[FEATURE_COLS], train_feat[TARGET]
X_val,   y_val   = val_feat[FEATURE_COLS],   val_feat[TARGET]

print("Training LightGBM...")
model = lgb.LGBMClassifier(
    objective="binary",
    n_estimators=500,
    learning_rate=0.05,
    num_leaves=63,
    scale_pos_weight=int((y_train == 0).sum() / (y_train == 1).sum()),
    colsample_bytree=0.8,
    subsample=0.8,
    min_child_samples=50,
    random_state=42,
    n_jobs=-1,
)
model.fit(X_train, y_train,
          eval_set=[(X_val, y_val)],
          callbacks=[lgb.early_stopping(30), lgb.log_evaluation(50)])

print("Calibrating with Platt scaling...")
calibrated = CalibratedClassifierCV(model, method="sigmoid", cv="prefit")
calibrated.fit(X_val, y_val)

val_probs = calibrated.predict_proba(X_val)[:, 1]
print(f"Val AUC:      {roc_auc_score(y_val, val_probs):.4f}")
print(f"Val Log-loss: {log_loss(y_val, val_probs):.4f}")

print("Exporting to ONNX...")
from onnxmltools import convert_lightgbm
from onnxmltools.convert.common.data_types import FloatTensorType
import onnx

initial_type = [("float_input", FloatTensorType([None, len(FEATURE_COLS)]))]
onnx_model = convert_lightgbm(model.booster_, initial_types=initial_type,
                               target_opset=12)
onnx.save_model(onnx_model, "triton/models/propensity/1/model.onnx")
pickle.dump(calibrated, open("encoders/calibrator.pkl", "wb"))
print("Done.")
```

### Triton Model Config (`triton/models/propensity/config.pbtxt`)

```protobuf
name: "propensity"
backend: "onnxruntime"
max_batch_size: 128

input [
  {
    name: "float_input"
    data_type: TYPE_FP32
    dims: [ 13 ]
  }
]

output [
  {
    name: "probabilities"
    data_type: TYPE_FP32
    dims: [ 2 ]
  }
]

dynamic_batching {
  preferred_batch_size: [ 16, 32, 64 ]
  max_queue_delay_microseconds: 1000
}
```

---

## 8. Redis Schema

### 8.1 Key Layout

```
# Device feature hash
HSET device:{device_id}
  hist_ctr          0.032    # per-device CTR from training set
  device_type       0        # raw Avazu int
  device_conn_type  2        # raw Avazu int
  C1                4        # label-encoded int

# App feature hash
HSET app:{app_id}
  hist_ctr          0.018    # per-app CTR from training set
  app_category      7        # label-encoded int
  site_category     3        # label-encoded int

# Real-time sliding-window EMA (updated by Kafka consumer, 24h TTL)
SET  device:{device_id}:recent_ctr  0.041
SET  app:{app_id}:recent_ctr        0.022
```

### 8.2 Redis Seed Script (`offline/seed_redis.py`)

```python
import pandas as pd
import pickle
import redis
from avazu_features import load_encoders, compute_ctr_stats, apply_encoders

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

df       = pd.read_csv("data/avazu_2m.csv",
                        dtype={"device_id": str, "app_id": str,
                               "site_category": str, "app_category": str, "C1": str})
train_df = df.iloc[:int(len(df) * 0.8)].copy()

encoders   = load_encoders()
device_ctr = pickle.load(open("encoders/device_ctr.pkl", "rb"))
app_ctr    = pickle.load(open("encoders/app_ctr.pkl",    "rb"))
train_df   = apply_encoders(train_df, encoders)

# Seed device features
print("Seeding device features...")
device_stats = train_df.groupby("device_id").agg(
    hist_ctr         = ("click",            "mean"),
    device_type      = ("device_type",      "first"),
    device_conn_type = ("device_conn_type", "first"),
    C1               = ("C1",               "first"),
).reset_index()

pipe = r.pipeline(transaction=False)
for _, row in device_stats.iterrows():
    key = f"device:{row['device_id']}"
    pipe.hset(key, mapping={
        "hist_ctr":          round(float(row["hist_ctr"]), 6),
        "device_type":       int(row["device_type"]),
        "device_conn_type":  int(row["device_conn_type"]),
        "C1":                int(row["C1"]),
    })
    pipe.setex(f"{key}:recent_ctr", 86400, round(float(row["hist_ctr"]), 6))
    if len(pipe) >= 500:
        pipe.execute()
        pipe = r.pipeline(transaction=False)
pipe.execute()
print(f"  {len(device_stats):,} devices seeded")

# Seed app features
print("Seeding app features...")
app_stats = train_df.groupby("app_id").agg(
    hist_ctr      = ("click",        "mean"),
    app_category  = ("app_category", "first"),
    site_category = ("site_category","first"),
).reset_index()

pipe = r.pipeline(transaction=False)
for _, row in app_stats.iterrows():
    key = f"app:{row['app_id']}"
    pipe.hset(key, mapping={
        "hist_ctr":      round(float(row["hist_ctr"]), 6),
        "app_category":  int(row["app_category"]),
        "site_category": int(row["site_category"]),
    })
    pipe.setex(f"{key}:recent_ctr", 86400, round(float(row["hist_ctr"]), 6))
    if len(pipe) >= 500:
        pipe.execute()
        pipe = r.pipeline(transaction=False)
pipe.execute()
print(f"  {len(app_stats):,} apps seeded")
print("Redis seed complete.")
```

---

## 9. Backend API

### 9.1 Endpoints

| Method | Path | Description |
|---|---|---|
| POST | `/recommend` | Score candidate apps for a device, return ranked list |
| POST | `/click` | Log click event to Kafka |
| GET | `/health` | Liveness check |

### 9.2 Feature Assembly (`api/features.py`)

```python
import redis.asyncio as aioredis
import numpy as np
from datetime import datetime

GLOBAL_DEVICE_CTR = 0.170   # Avazu global CTR ~17%
GLOBAL_APP_CTR    = 0.170

async def build_feature_vectors(
    r: aioredis.Redis,
    device_id: str,
    app_ids: list[str],
    banner_pos: int,
    site_category_enc: int,
) -> np.ndarray:

    now         = datetime.utcnow()
    hour_of_day = now.hour
    day_of_week = now.weekday()

    device_data  = await r.hgetall(f"device:{device_id}")
    device_rctr  = await r.get(f"device:{device_id}:recent_ctr")

    device_hist_ctr   = float(device_data.get(b"hist_ctr",         GLOBAL_DEVICE_CTR))
    device_recent_ctr = float(device_rctr  or device_hist_ctr)
    device_type       = int(device_data.get(b"device_type",       0))
    device_conn_type  = int(device_data.get(b"device_conn_type",  0))
    c1                = int(device_data.get(b"C1",                 0))

    pipe = r.pipeline()
    for app_id in app_ids:
        pipe.hgetall(f"app:{app_id}")
        pipe.get(f"app:{app_id}:recent_ctr")
    app_results = await pipe.execute()

    rows = []
    for i, app_id in enumerate(app_ids):
        app_data       = app_results[i * 2]
        app_rctr       = app_results[i * 2 + 1]
        app_hist_ctr   = float(app_data.get(b"hist_ctr",     GLOBAL_APP_CTR))
        app_recent_ctr = float(app_rctr  or app_hist_ctr)
        app_category   = int(app_data.get(b"app_category", 0))

        rows.append([
            device_hist_ctr,
            device_recent_ctr,
            app_hist_ctr,
            app_recent_ctr,
            banner_pos,
            site_category_enc,
            app_category,
            device_type,
            device_conn_type,
            hour_of_day,
            day_of_week,
            c1,
            device_type * 100 + app_category,   # device_x_app_cat
        ])

    return np.array(rows, dtype=np.float32)
```

### 9.3 Recommend Endpoint (`api/main.py` excerpt)

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager
import redis.asyncio as aioredis
import uuid
from datetime import datetime

from .features      import build_feature_vectors
from .triton_client import TritonClient
from .kafka_producer import KafkaProducer
from .schemas        import RecommendRequest, RecommendResponse, ScoredApp, ClickEvent

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.redis    = aioredis.from_url("redis://redis:6379")
    app.state.triton   = TritonClient("http://triton:8000")
    app.state.producer = KafkaProducer("kafka:9092")
    yield
    await app.state.redis.aclose()

app = FastAPI(lifespan=lifespan)

@app.post("/recommend", response_model=RecommendResponse)
async def recommend(req: RecommendRequest):
    banner_pos        = req.context.get("banner_pos", 0)
    site_category_enc = req.context.get("site_category_enc", 0)

    features = await build_feature_vectors(
        app.state.redis, req.device_id, req.candidate_app_ids,
        banner_pos, site_category_enc,
    )
    probs = await app.state.triton.score(features)

    scored = sorted(
        [ScoredApp(app_id=aid, p_click=round(p, 4))
         for aid, p in zip(req.candidate_app_ids, probs)],
        key=lambda x: x.p_click, reverse=True
    )

    imp_id = str(uuid.uuid4())
    for rank, item in enumerate(scored, 1):
        await app.state.producer.send("impressions", key=req.device_id, value={
            "impression_id": imp_id,
            "device_id":     req.device_id,
            "app_id":        item.app_id,
            "position":      rank,
            "p_click":       item.p_click,
            "hour":          datetime.utcnow().strftime("%y%m%d%H"),
            "ts":            datetime.utcnow().isoformat(),
        })

    return RecommendResponse(device_id=req.device_id, ranked_apps=scored)

@app.post("/click")
async def click(event: ClickEvent):
    await app.state.producer.send("clicks", key=event.device_id, value={
        "device_id":     event.device_id,
        "app_id":        event.app_id,
        "impression_id": event.impression_id,
        "ts":            datetime.utcnow().isoformat(),
    })
    return {"status": "ok"}
```

### 9.4 Triton Client (`api/triton_client.py`)

```python
import httpx
import numpy as np

class TritonClient:
    def __init__(self, url: str):
        self.infer_url = f"{url}/v2/models/propensity/infer"

    async def score(self, features: np.ndarray) -> list[float]:
        payload = {
            "inputs": [{
                "name":     "float_input",
                "shape":    list(features.shape),
                "datatype": "FP32",
                "data":     features.flatten().tolist(),
            }]
        }
        async with httpx.AsyncClient(timeout=2.0) as client:
            resp = await client.post(self.infer_url, json=payload)
            resp.raise_for_status()
        raw = resp.json()["outputs"][0]["data"]
        n   = features.shape[0]
        return [raw[i * 2 + 1] for i in range(n)]   # p_click column
```

---

## 10. Kafka Event Consumer (`consumer/worker.py`)

```python
from confluent_kafka import Consumer
import redis, json

r = redis.Redis.from_url("redis://redis:6379", decode_responses=True)

consumer = Consumer({
    "bootstrap.servers": "kafka:9092",
    "group.id":          "avazu-ctr-updater",
    "auto.offset.reset": "earliest",
})
consumer.subscribe(["impressions", "clicks"])

WINDOW = 1000   # EMA window

def ema_update(key: str, clicked: int, ttl: int = 86400):
    current = float(r.get(key) or 0.170)   # Avazu global CTR fallback
    alpha   = 2.0 / (WINDOW + 1)
    updated = alpha * clicked + (1 - alpha) * current
    r.setex(key, ttl, round(updated, 6))

print("Consumer started...")
while True:
    msg = consumer.poll(1.0)
    if msg is None or msg.error():
        continue
    try:
        event = json.loads(msg.value())
        topic = msg.topic()
        dk = f"device:{event['device_id']}:recent_ctr"
        ak = f"app:{event['app_id']}:recent_ctr"
        if topic == "impressions":
            ema_update(dk, 0); ema_update(ak, 0)
        elif topic == "clicks":
            ema_update(dk, 1); ema_update(ak, 1)
    except (KeyError, json.JSONDecodeError) as e:
        print(f"Bad event: {e}")
```

---

## 11. React Frontend

### 11.1 Component Tree

```
App
├── DeviceSelector       -- dropdown of device_ids sampled from Avazu val set
├── AdGrid               -- calls /recommend, renders ranked AdCard list
│   └── AdCard           -- shows app_id, app_category, banner_pos, p(click) badge
│       └── ScoreBadge   -- green >=0.20 | amber 0.10-0.20 | red <0.10
│                           (thresholds tuned to Avazu global CTR ~17%)
└── EventLog             -- live feed of impression/click events from this session
```

### 11.2 API Client (`frontend/src/api/client.js`)

```javascript
const BASE = import.meta.env.VITE_API_BASE ?? "http://localhost:8080";

// Candidate apps: real app_ids sampled from Avazu val set
export const CANDIDATE_APPS = [
  { app_id: "ecad2386", app_category: "Games" },
  { app_id: "febd1138", app_category: "Sports" },
  { app_id: "1bdf0d3d", app_category: "Shopping" },
  { app_id: "85f751fd", app_category: "Travel" },
  { app_id: "fc6fa53d", app_category: "Utilities" },
  { app_id: "8ded1f7a", app_category: "Music" },
];

export async function getRecommendations(deviceId, bannerPos = 0, siteCatEnc = 0) {
  const res = await fetch(`${BASE}/recommend`, {
    method:  "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      device_id:         deviceId,
      candidate_app_ids: CANDIDATE_APPS.map(a => a.app_id),
      context: { banner_pos: bannerPos, site_category_enc: siteCatEnc },
    }),
  });
  if (!res.ok) throw new Error(`/recommend ${res.status}`);
  return res.json();
}

export async function logClick(deviceId, appId, impressionId) {
  await fetch(`${BASE}/click`, {
    method:  "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ device_id: deviceId, app_id: appId,
                           impression_id: impressionId }),
  });
}
```

### 11.3 AdCard Component (`frontend/src/components/AdCard.jsx`)

```jsx
import React from "react";

const badge = (p) => {
  if (p >= 0.20) return { bg: "#16a34a", label: "High" };
  if (p >= 0.10) return { bg: "#d97706", label: "Med"  };
  return               { bg: "#dc2626", label: "Low"  };
};

export default function AdCard({ app, onClickAd }) {
  const { bg, label } = badge(app.p_click);
  return (
    <div style={{ border: "1px solid #e5e7eb", borderRadius: 8,
                  padding: 16, display: "flex", flexDirection: "column", gap: 8 }}>
      <div style={{ fontWeight: 600, fontSize: 14, fontFamily: "monospace" }}>
        {app.app_id}
      </div>
      <div style={{ fontSize: 12, color: "#6b7280" }}>{app.app_category}</div>
      <div style={{ display: "inline-flex", alignItems: "center", gap: 6,
                    background: bg, color: "#fff", borderRadius: 4,
                    padding: "2px 8px", fontSize: 12, fontWeight: 500,
                    alignSelf: "flex-start" }}>
        <span>{label}</span>
        <span>p(click) = {app.p_click.toFixed(3)}</span>
      </div>
      <button onClick={() => onClickAd(app)}
              style={{ marginTop: 8, padding: "6px 12px", borderRadius: 6,
                       border: "1px solid #d1d5db", cursor: "pointer",
                       background: "#f9fafb" }}>
        Simulate click
      </button>
    </div>
  );
}
```

---

## 12. Data Flow — Request Lifecycle

```
User picks a device_id in DeviceSelector
          |
          v
React calls POST /recommend
{ device_id, candidate_app_ids: [6 app_ids], context: {banner_pos, site_category_enc} }
          |
          v
FastAPI: build_feature_vectors()
  -- Redis HGETALL device:{device_id}          (1 call)
  -- Redis GET     device:{device_id}:recent_ctr
  -- Redis pipeline HGETALL + GET per app_id   (1 pipeline for all 6 apps)
  -- assemble numpy float32 matrix  [6 x 13]
          |
          v
FastAPI: triton_client.score()
  -- POST /v2/models/propensity/infer
  -- Triton: ONNX forward pass [6 x 13] -> [6 x 2] probabilities
  -- extract p_click column -> [0.22, 0.14, 0.09, 0.08, 0.05, 0.03]
          |
          v
FastAPI: sort 6 apps by p(click) desc
  -- publish 6 impression events -> Kafka topic: impressions
  -- return ranked list to React
          |
          v
React renders AdGrid (apps sorted by p_click, colour-coded badges)
          |
          v
User clicks "Simulate click" on an AdCard
  -- React calls POST /click { device_id, app_id, impression_id }
  -- FastAPI publishes click event -> Kafka topic: clicks
          |
          v
Kafka Consumer (worker.py)
  -- impression events: EMA update device+app CTR with clicked=0
  -- click events:      EMA update device+app CTR with clicked=1
  -- Redis recent_ctr keys updated, 24h TTL refreshed
  -- next /recommend call for same device sees updated CTR features
```

---

## 13. Offline to Online Feature Consistency

| Feature | Training (`train.py`) | Serving (`features.py`) | Enforcement |
|---|---|---|---|
| `device_hist_ctr` | `groupby("device_id")["click"].mean()` | `Redis HGET device:{id} hist_ctr` | `seed_redis.py` writes training-computed values |
| `app_hist_ctr` | `groupby("app_id")["click"].mean()` | `Redis HGET app:{id} hist_ctr` | Same |
| `device_recent_ctr` | Equal to `device_hist_ctr` at train time | `Redis GET device:{id}:recent_ctr` | Init to hist_ctr; updated live by consumer |
| `site_category` | `LabelEncoder` fitted on training set | Integer in Redis `app:{id}` hash | `seed_redis.py` calls `apply_encoders()` before writing |
| `app_category` | `LabelEncoder` | Integer in Redis | Same |
| `C1` | `LabelEncoder` | Integer in Redis | Same |
| `device_x_app_cat` | `device_type * 100 + app_category` | Same formula in `features.py` | Single formula, code review |
| `hour_of_day` | Extracted from Avazu `hour` int | `datetime.utcnow().hour` | No encoder needed |
| `day_of_week` | `datetime.strptime(s[:6]).weekday()` | `datetime.utcnow().weekday()` | No encoder needed |
| Unknown device/app | Falls back to global CTR mean | `GLOBAL_DEVICE_CTR = 0.170` constant | Constant must match training global mean |

**Golden rule:** `encoders/label_encoders.pkl`, `encoders/device_ctr.pkl`, and `encoders/app_ctr.pkl` are computed once by `train.py` and consumed by both `seed_redis.py` and the API. Never recompute them independently.

---

## 14. Docker Compose

```yaml
version: "3.9"

services:

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports: ["2181:2181"]

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on: [zookeeper]
    ports: ["9092:9092"]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    command: redis-server --save "" --appendonly no

  triton:
    image: nvcr.io/nvidia/tritonserver:23.10-py3-min
    ports:
      - "8000:8000"
      - "8001:8001"
      - "8002:8002"
    volumes:
      - ./triton/models:/models
    command: >
      tritonserver --model-repository=/models
      --backend-config=onnxruntime,execution_accelerator=cpu

  api:
    build: ./api
    ports: ["8080:8080"]
    depends_on: [redis, kafka, triton]
    environment:
      REDIS_URL:       redis://redis:6379
      KAFKA_BOOTSTRAP: kafka:9092
      TRITON_URL:      http://triton:8000

  consumer:
    build: ./consumer
    depends_on: [kafka, redis]
    environment:
      KAFKA_BOOTSTRAP: kafka:9092
      REDIS_URL:       redis://redis:6379

  frontend:
    build: ./frontend
    ports: ["3000:3000"]
    depends_on: [api]
    environment:
      VITE_API_BASE: http://localhost:8080
```

---

## 15. Weekend Build Plan

### Saturday — Data, Training, Infra

| Time | Task |
|---|---|
| 09:00–09:30 | Download Avazu from Kaggle; run `prep.py` to produce `avazu_2m.csv` |
| 09:30–10:30 | Write `docker-compose.yml`; `docker-compose up -d`; verify all services healthy |
| 10:30–12:00 | Write `avazu_features.py` (shared module); write and run `train.py` |
| 12:00–13:00 | Lunch |
| 13:00–14:00 | Verify ONNX export; restart Triton; `curl /v2/models/propensity/ready` |
| 14:00–16:00 | Write `seed_redis.py`; run it; spot-check device + app keys in `redis-cli` |
| 16:00–18:00 | Write Kafka producer + consumer; test EMA CTR update end-to-end |

### Sunday — API + Frontend + Integration

| Time | Task |
|---|---|
| 09:00–11:00 | Build FastAPI: `main.py`, `features.py`, `triton_client.py`, `schemas.py` |
| 11:00–12:30 | Test `/recommend` with curl; verify 13-feature vector matches training |
| 12:30–13:30 | Lunch |
| 13:30–15:30 | Build React: `DeviceSelector`, `AdGrid`, `AdCard`, `EventLog` |
| 15:30–17:00 | Wire frontend to API; test full click flow in browser |
| 17:00–18:00 | Verify Redis CTR updates after clicks; smoke test full system; record demo |

---

## 16. Cost Analysis

| Service | Local (Docker Compose) | Cloud Free Tier |
|---|---|---|
| Kafka | $0 — container | Confluent Cloud free 10GB/month |
| Redis | $0 — container | Redis Cloud free 30MB |
| Triton | $0 — CPU container | N/A (no free tier) |
| FastAPI | $0 — container | Railway / Render free tier |
| React | $0 — Vite dev server | Vercel / Netlify free tier |
| **Total** | **$0** | **~$0** |

---

## 17. Known Simplifications vs Full Production

| Concern | This Design | Production Gap |
|---|---|---|
| Kafka | Single broker, no replication | Multi-broker, ISR=2, replication factor 3 |
| Triton | CPU inference | GPU instance with TensorRT optimisation |
| Redis | Standalone | Redis Cluster or Elasticache |
| Feature freshness | EMA CTR via consumer | Real-time aggregation with Flink / Spark Streaming |
| Position bias | `banner_pos` is a raw feature; bias not corrected | IPW correction or learnable position embedding |
| Cold start | Falls back to global Avazu CTR (17%) | Separate content-based or popularity ranker |
| Retraining | Manual re-run of `train.py` | Scheduled Airflow DAG, triggered by data drift |
| Auth | None | API gateway with JWT |
| A/B testing | Not implemented | Experimentation framework (Statsig, LaunchDarkly) |
| Model versioning | File on disk | MLflow or Weights & Biases |
| Avazu C14–C21 | Dropped for brevity | Include as additional categorical embeddings |
| `device_ip` | Dropped (too high cardinality) | Geo-IP lookup → country/city/region features |

---

## 18. Getting Started

```bash
# 0. Download Avazu dataset
kaggle competitions download -c avazu-ctr-prediction -p data/
# Or 50K sample for rapid prototyping:
# kaggle datasets download -d gauravduttakiit/avazu-ctr-prediction-with-random-50k-rows -p data/

# 1. Prepare 2M sample
python offline/prep.py

# 2. Install Python dependencies
pip install lightgbm scikit-learn onnxmltools onnx pandas numpy \
            redis confluent-kafka fastapi uvicorn httpx

# 3. Create output dirs
mkdir -p encoders triton/models/propensity/1

# 4. Bring up all services
docker-compose up -d

# 5. Train model (writes encoders/ and triton/models/propensity/1/model.onnx)
python offline/train.py

# 6. Restart Triton to pick up the new ONNX model
docker-compose restart triton

# 7. Verify Triton loaded the model
curl http://localhost:8000/v2/models/propensity/ready

# 8. Seed Redis with device + app features
python offline/seed_redis.py

# 9. Start the API
uvicorn api.main:app --reload --port 8080

# 10. Start React frontend
cd frontend && npm install && npm run dev

# 11. Open http://localhost:3000
#     Pick a device_id, load ranked ads, simulate clicks,
#     watch Redis recent_ctr update live.
```

---

*All simplifications are intentional weekend tradeoffs documented in Section 17. The `avazu_features.py` shared module is the single source of truth that prevents training-serving skew. The core serving path — Avazu-derived features in Redis → FastAPI → Triton ONNX → Kafka → consumer CTR update — is production-equivalent.*
