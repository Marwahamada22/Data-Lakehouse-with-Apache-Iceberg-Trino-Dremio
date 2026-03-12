# 🏗️ Data Lakehouse with Apache Iceberg, Trino & Dremio

A fully containerized Data Lakehouse built on the **Medallion Architecture** (Bronze → Silver → Gold) using Apache Iceberg as the table format, Nessie as the catalog, MinIO as object storage, and Trino  ,  Dremio as the query engine.

---

## 🧱 Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Bronze    │────▶│   Silver    │────▶│    Gold     │
│  (Raw Data) │     │  (Cleaned)  │     │ (Aggregated)│
└─────────────┘     └─────────────┘     └─────────────┘
        │                  │                   │
        └──────────────────┴───────────────────┘
                           │
                    ┌──────▼──────┐
                    │   Iceberg   │  ← Table Format
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │    Nessie   │  ← Catalog (Version Control)
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │    MinIO    │  ← Object Storage (S3-compatible)
                    └─────────────┘

        Trino ──────────────────────── Query Engine
        Dremio ─────────────────────── BI + Semantic Layer
```

---

## 🛠️ Stack

| Component | Version | Role | Port |
|-----------|---------|------|------|
| Apache Iceberg | - | Table Format | - |
| Nessie | 0.99.0 | Iceberg Catalog + Git-like versioning | 19120 |
| MinIO | 2024-05-10 | S3-compatible Object Storage | 9000 / 9001 |
| Trino | 450 | Distributed SQL Query Engine | 8080 |
| Dremio OSS | 24.3 | BI Layer + Semantic Layer | 9047 |
| PostgreSQL | 14 | Nessie metadata backend | 5436 |

---

## 📁 Project Structure

```
bigdata-ecosystem-sandbox/
├── docker-compose.yml
└── mnt/
    ├── trino/
    │   └── etc/
    │       ├── config.properties
    │       ├── node.properties
    │       ├── jvm.config
    │       └── catalog/
    │           └── iceberg.properties
    ├── minio/               ← Parquet files + Iceberg metadata
    ├── nessie-data/         ← Nessie version store
    └── postgres/            ← Nessie DB backend
```

---

## ⚙️ Trino Config Files

### `etc/config.properties`
```properties
coordinator=true
node-scheduler.include-coordinator=true
http-server.http.port=8080
discovery.uri=http://localhost:8080
```

### `etc/node.properties`
```properties
node.environment=production
node.id=trino-node-1
node.data-dir=/var/trino/data
```

### `etc/jvm.config`
```
-server
-Xmx4G
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
```

### `etc/catalog/iceberg.properties`
```properties
connector.name=iceberg
iceberg.catalog.type=rest
iceberg.rest-catalog.uri=http://nessie:19120/iceberg
iceberg.rest-catalog.warehouse=s3://warehouse/
iceberg.rest-catalog.security=NONE
fs.native-s3.enabled=true
s3.endpoint=http://minio:9000
s3.region=us-east-1
s3.path-style-access=true
s3.aws-access-key=minioadmin
s3.aws-secret-key=minioadmin
```

---

## 🚀 Getting Started

### 1. Start Services
```bash
docker-compose up -d postgres minio nessie trino dremio
```

### 2. Launch Trino CLI
```bash
# Download CLI (first time only)
wget https://repo1.maven.org/maven2/io/trino/trino-cli/450/trino-cli-450-executable.jar \
  -O ~/trino-cli.jar
chmod +x ~/trino-cli.jar

# Connect
java -jar ~/trino-cli.jar --server http://localhost:8080
```

### 3. Verify Setup
```sql
SHOW CATALOGS;
-- Expected: iceberg, system
```

---

## 🥉 Bronze Layer — Raw Data

```sql
CREATE SCHEMA IF NOT EXISTS iceberg.bronze
WITH (location = 's3://warehouse/bronze');

CREATE TABLE iceberg.bronze.orders (
    order_id     BIGINT,
    customer     VARCHAR,
    product      VARCHAR,
    amount       DOUBLE,
    status       VARCHAR,
    order_date   DATE,
    country      VARCHAR
)
WITH (format = 'PARQUET', partitioning = ARRAY['country']);

INSERT INTO iceberg.bronze.orders VALUES
(1,  'marwa',   'Laptop',  1200.0, 'completed', DATE '2024-01-05', 'EG'),
(2,  'manar',   'Phone',    800.0, 'completed', DATE '2024-01-10', 'SA'),
(3,  'sama',    'Tablet',   450.0, 'cancelled', DATE '2024-01-15', 'EG'),
(4,  'ziad',    'Laptop',  1200.0, 'completed', DATE '2024-02-01', 'AE'),
(5,  'hamza',   'Phone',    800.0, 'completed', DATE '2024-02-10', 'SA'),
(6,  'Layla',   'Headset',  150.0, 'cancelled', DATE '2024-02-14', 'EG'),
(7,  'Youssef', 'Tablet',   450.0, 'completed', DATE '2024-03-01', 'AE'),
(8,  'Nour',    'Laptop',  1200.0, 'completed', DATE '2024-03-15', 'SA'),
(9,  'Khaled',  'Headset',  150.0, 'completed', DATE '2024-03-20', 'EG'),
(10, 'Mariam',  'Phone',    800.0, 'cancelled', DATE '2024-03-25', 'AE');
```

---

## 🥈 Silver Layer — Cleaned Data

```sql
CREATE SCHEMA IF NOT EXISTS iceberg.silver
WITH (location = 's3://warehouse/silver');

CREATE TABLE iceberg.silver.orders_clean
WITH (format = 'PARQUET', partitioning = ARRAY['country'])
AS
SELECT
    order_id,
    customer,
    product,
    amount,
    order_date,
    country
FROM iceberg.bronze.orders
WHERE status = 'completed';
```

**Result:** 10 rows → 7 rows (cancelled orders removed)

---

## 🥇 Gold Layer — Aggregated Data

```sql
CREATE SCHEMA IF NOT EXISTS iceberg.gold
WITH (location = 's3://warehouse/gold');

-- Revenue by Country
CREATE TABLE iceberg.gold.revenue_by_country
WITH (format = 'PARQUET')
AS
SELECT
    country,
    COUNT(order_id)  AS total_orders,
    SUM(amount)      AS total_revenue,
    AVG(amount)      AS avg_order_value
FROM iceberg.silver.orders_clean
GROUP BY country;

-- Revenue by Product
CREATE TABLE iceberg.gold.revenue_by_product
WITH (format = 'PARQUET')
AS
SELECT
    product,
    COUNT(order_id)  AS total_orders,
    SUM(amount)      AS total_revenue
FROM iceberg.silver.orders_clean
GROUP BY product
ORDER BY total_revenue DESC;
```

---

## 📊 Sample Results

### Revenue by Country
| country | total_orders | total_revenue | avg_order_value |
|---------|-------------|---------------|-----------------|
| SA | 3 | 2800.0 | 933.33 |
| AE | 2 | 1650.0 | 825.0 |
| EG | 2 | 1350.0 | 675.0 |

### Revenue by Product
| product | total_orders | total_revenue |
|---------|-------------|---------------|
| Laptop | 3 | 3600.0 |
| Phone | 2 | 1600.0 |
| Tablet | 1 | 450.0 |
| Headset | 1 | 150.0 |

---

## 🔍 Useful Queries

```sql
-- Show all schemas
SHOW SCHEMAS IN iceberg;

-- Show all tables per layer
SHOW TABLES IN iceberg.bronze;
SHOW TABLES IN iceberg.silver;
SHOW TABLES IN iceberg.gold;

-- Check Iceberg table metadata
SELECT * FROM iceberg.bronze."orders$snapshots";

-- Time Travel
SELECT * FROM iceberg.bronze.orders
FOR VERSION AS OF <snapshot_id>;
```

---

## 🔗 Service URLs

| Service | URL | Credentials |
|---------|-----|-------------|
| Trino UI | http://localhost:8080 | any username |
| MinIO Console | http://localhost:9001 | minioadmin / minioadmin |
| Nessie API | http://localhost:19120 | - |
| Dremio UI | http://localhost:9047 | admin account |


|
