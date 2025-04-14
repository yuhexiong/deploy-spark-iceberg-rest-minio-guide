# Deploy Spark Iceberg REST MinIO Guide

建構 Lakehouse 架構，透過 Iceberg 提供強大的 schema 演進、ACID 交易、時間旅行以及多引擎相容性，讓資料湖具備資料倉儲等級的管理與查詢能力。

本指南提供 Spark + Iceberg REST + MinIO 的部署方式，並整合 Apache Doris，涵蓋環境設定、SQL 操作與無需 Schema 的資料搬移功能。


## Overview
- 表格格式：Iceberg v1.8.1  
- 計算引擎：Spark v3.5.2  
- 資料庫：Doris v2.1.1.8  

## Architecture

### 1️\. Compute Layer: Spark-Iceberg  
  - 負責處理查詢與資料操作  
  - 仰賴 Iceberg REST 進行中繼資料管理  
  - 讀寫 MinIO 儲存的資料  

### 2️\. Service Layer: Iceberg REST  
  - 管理 Iceberg 資料表的中繼資料  
  - 與 MinIO 互動以儲存 Iceberg 的資料  

### 3️\. Storage Layer: MinIO  
  - 類似 S3 的物件儲存服務  
  - 儲存 Iceberg 的資料與中繼資料  

### 4️\. Physical Storage Layer: Disk/Cloud Storage  
  - 最終資料儲存位置  
  - MinIO 透過 Volume 存取實體儲存空間  


## Run

### Run
```bash
docker exec -it spark-iceberg /bin/bash
```

### Execute SQL in Spark
```bash
spark-sql --packages org.apache.iceberg:iceberg-spark-runtime-3.5_2.12:1.8.1 \
    --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions \
    --conf spark.sql.catalog.spark_catalog=org.apache.iceberg.spark.SparkSessionCatalog \
    --conf spark.sql.catalog.spark_catalog.type=hive \
    --conf spark.sql.catalog.local=org.apache.iceberg.spark.SparkCatalog \
    --conf spark.sql.catalog.local.type=hadoop \
    --conf spark.sql.catalog.local.warehouse=/home/iceberg/warehouse \
    --conf spark.sql.defaultCatalog=local
```

### Create Table
```sql
CREATE TABLE demo.nyc.taxis
(
  vendor_id bigint,
  trip_id bigint,
  trip_distance float,
  fare_amount double,
  store_and_fwd_flag string
)
PARTITIONED BY (vendor_id);
```

### Insert Data
```sql
INSERT INTO demo.nyc.taxis
VALUES (1, 1000371, 1.8, 15.32, 'N'), (2, 1000372, 2.5, 22.15, 'N'), (2, 1000373, 0.9, 9.01, 'N'), (1, 1000374, 8.4, 42.13, 'Y');
```


### Select Data
```sql
SELECT * FROM demo.nyc.taxis;
```

## Doris

Connect to Iceberg from Doris.

### See All Catalogs
```sql
SELECT * FROM CATALOGS();
```

### Drop Catalog (if you need to)
```sql
DROP CATALOG IF EXISTS iceberg_catalog;
```

### Create Catalog
`aws.region` 是無效參數。

```sql
CREATE CATALOG iceberg_catalog
PROPERTIES (
    "type"="iceberg",
    "iceberg.catalog.type"="rest",
    "uri"="{ICEBERG_IP}:8181",
    "s3.endpoint"="{ICEBERG_IP}:9000",
    "s3.access_key"="admin",
    "s3.secret_key"="password",
    "s3.region"="us-east-1"
);
```

### Show Databases in catalog
```sql
SHOW DATABASES FROM iceberg_catalog;
```

### Show Tables in catalog.database
```sql
SHOW TABLES FROM iceberg_catalog.nyc;
```

### Show Data in catalog.database.table
```sql
SELECT * FROM iceberg_catalog.nyc.taxis;
```

### Move Data
將 Doris 的資料表搬移到 [catalog].[database].[new_table_name]，無需預先建立 Schema。
```sql
CREATE TABLE iceberg_catalog.database.table AS SELECT * FROM database.table;
```



## References
- [Apache Iceberg](https://iceberg.apache.org/spark-quickstart/#docker-compose)  
- [Apache Doris -  Repository](https://doris.apache.org/docs/3.0/sql-manual/sql-statements/data-modification/backup-and-restore/CREATE-REPOSITORY#examples) 
- [Apache Doris - Iceberg](https://doris.apache.org/docs/lakehouse/lakehouse-best-practices/doris-iceberg)


