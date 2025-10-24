# Yelp-at-Scale — Spark + PostgreSQL (Containerized)

End-to-end pipeline to ingest the **Yelp Open Dataset**, run scalable analytics with **Apache Spark**, and persist/serve results from **PostgreSQL**. Everything runs in Docker.

---

## Contents

* `notebooks/01_ingest_yelp.ipynb` — Ingest + basic seeding to Postgres
* `notebooks/02_queries.ipynb` — Transformations, analytics (q1–q4), CSV exports, Postgres writes, CRUD demo
* `docker-compose.yml` — Jupyter + Postgres services
* `data/` — Local data folder (raw JSON, parquet, csv outputs)
* `scripts/` — Helper shell snippets (optional)
* `diagram.drawio` — Dataflow diagram (optional)

---

## Prerequisites

* Docker & Docker Compose
* ~15–20 GB free disk space
* (Optional) Git

---

## Dataset

**Yelp Open Dataset (academic)**
Kaggle: [https://www.kaggle.com/datasets/yelp-dataset/yelp-dataset](https://www.kaggle.com/datasets/yelp-dataset/yelp-dataset)

Download the dataset and prepare the **5 JSON** files:

```
yelp_academic_dataset_business.json
yelp_academic_dataset_checkin.json
yelp_academic_dataset_review.json
yelp_academic_dataset_tip.json
yelp_academic_dataset_user.json
```

> ⚠️ Some downloads arrive as **ZIP data but named `.json`** (header starts with `504b0304`). If that happens, rename to `.zip` and extract the real `.json`.

Example (run from repo root):

```bash
mkdir -p data/raw
# put the 5 files into data/raw first

cd data/raw
for f in yelp_academic_dataset_*.json; do
  if [ "$(od -An -tx1 -N4 "$f" | tr -d ' \n')" = "504b0304" ]; then
    mv "$f" "$f.zip"
    unzip -p "$f.zip" > "${f%.zip}"
    rm "$f.zip"
  fi
done

# quick sanity
wc -l yelp_academic_dataset_*.json
head -1 yelp_academic_dataset_business.json
```

---

## Quick Start

```bash
# 1) Clone & enter
git clone https://github.com/PeakFiction/INFSIndividualProject.git
cd INFSIndividualProject

# 2) Start containers
docker compose up -d    # starts Postgres and Jupyter

# 3) Open Jupyter in your browser
#   http://localhost:8888  (check logs for the token if needed)
docker logs yelp-jupyter | grep -m1 -E "http://.*token="
```

### 4) Run the notebooks (in order)

1. **`notebooks/01_ingest_yelp.ipynb`**

   * Reads **business** + **tip** JSON → writes Parquet (`data/parquet/`)
   * Seeds Postgres with:

     * `public.business` (thin version of business table)
     * `public.tip_small` (sample 100k tips)
     * `public.spark_ping` (sanity)
   * Validates counts

2. **`notebooks/02_queries.ipynb`**

   * Loads all 5 raw JSONs + parquet business/tip
   * Creates helper views (`business_category`, `checkin_counts`, `yelp_user_flag`)
   * Computes analytics queries **q1–q4**
   * Writes:

     * Parquet: `data/parquet/q{1..4}`
     * CSV: `data/csv/q{1..4}.csv`
     * Postgres tables: `public.q1`…`public.q4`
   * Builds indexes for snappy querying
   * **CRUD demo** against `q1` (INSERT/SELECT/UPDATE/DELETE)

### 5) Verify outputs

```bash
# CSVs
ls -lh data/csv/*.csv
head -3 data/csv/q1.csv

# Postgres
docker exec -it yelp-db psql -U postgres -d yelp -c "\dt+"
docker exec -it yelp-db psql -U postgres -d yelp -c "SELECT COUNT(*) FROM q1;"
```

---

## What the 4 Queries Produce

* **Q1: Top categories by quality**

  * For each category: average star rating (across businesses), total reviews, number of businesses
* **Q2: City × elite vs non-elite**

  * For each (city, state, is_elite): review count, avg stars, and star variability
* **Q3: Check-in intensity vs quality**

  * Businesses with ≥1 check-in and ≥8 reviews: check-in event count, average rating, review count
* **Q4: Open vs closed by category**

  * For each category and open/closed flag: counts, average rating, total reviews

---

## Re-running / Clean Slate

You can practice the demo repeatedly.

```bash
# Stop everything
docker compose down

# (Optional) Nuke derived data
rm -rf data/parquet/q* data/parquet/_tmp_q* data/csv/*.csv

# (Optional) Reset analytics tables in Postgres
docker exec -it yelp-db psql -U postgres -d yelp -c \
  "DROP TABLE IF EXISTS q1,q2,q3,q4 CASCADE;"

# Start again
docker compose up -d
```

---

## Demo Script (2–5 minutes)

1. Show **dataset** files in `data/raw`.
2. Open **01_ingest_yelp.ipynb**:

   * Run all; point out Postgres seed tables & row counts.
3. Open **02_queries.ipynb**:

   * Run the sanity checks (row counts, categories present, join reachability).
   * Execute **q1–q4**; show `show(20)` outputs.
   * Create CSVs and Postgres tables; verify with `\dt+` and quick `SELECT`.
   * Run the **CRUD** cell (insert/select/update/delete on `q1`).
4. Open a CSV (e.g., `data/csv/q1.csv`) and show first lines.

---

## Troubleshooting

* **JDBC driver not found**
  The notebooks load `postgresql-42.7.4.jar` from `/home/jovyan/jars/`.
  If you see `ClassNotFoundException: org.postgresql.Driver`, ensure:

  * File exists in the Jupyter container: `/home/jovyan/jars/postgresql-42.7.4.jar`
  * Notebook `SparkSession.builder` includes:

    ```
    .config("spark.jars", "/home/jovyan/jars/postgresql-42.7.4.jar")
    .config("spark.driver.extraClassPath", "/home/jovyan/jars/postgresql-42.7.4.jar")
    .config("spark.executor.extraClassPath", "/home/jovyan/jars/postgresql-42.7.4.jar")
    ```

* **Jupyter token**
  Get it with:

  ```
  docker logs yelp-jupyter | grep -m1 -E "http://.*token="
  ```

---

## License & Attribution

* **Data:** © Yelp — see the license bundled with the **Yelp Open Dataset** on Kaggle. Use is for academic/research only.
* **Code:** MIT (if you choose; update as needed in your repo).

---

## Citation (optional)

If you publish the results, cite the Yelp Open Dataset and acknowledge Spark & PostgreSQL.

```text
Yelp Open Dataset. Yelp Inc. https://www.yelp.com/dataset

Original Dataset:
https://www.kaggle.com/datasets/yelp-dataset/yelp-dataset

Apache Spark. https://spark.apache.org/
PostgreSQL. https://www.postgresql.org/
```

---

## Maintainer

* PeakFiction (GitHub): [https://github.com/PeakFiction](https://github.com/PeakFiction)
