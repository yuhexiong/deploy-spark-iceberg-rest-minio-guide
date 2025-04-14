# Deploy Spark Iceberg REST MinIO Guide

**(also provided Traditional Chinese version document [README-CH.md](README-CH.md).)**


Build a Lakehouse architecture with Iceberg, which provides powerful schema evolution, ACID transactions, time travel, and multi-engine compatibility, enabling the data lake to have data warehouse-level management and query capabilities.  

Provides a deployment guide for Spark + Iceberg REST + MinIO, integrating Apache Doris, covering setup, SQL operations, and schema-free data migration.  


## Overview
- Table Format: Iceberg v1.8.1
- Compute Engine: Spark v3.5.2
- Database: Doris v2.1.1.8

## Architecture

### 1\. Compute Layer: Spark-Iceberg  
  - Responsible for handling queries and data operations  
  - Relies on Iceberg REST for metadata management  
  - Reads and writes data to MinIO storage  

### 2️\. Service Layer: Iceberg REST  
  - Manages metadata for Iceberg tables  
  - Interacts with MinIO to store Iceberg data  

### 3️\. Storage Layer: MinIO  
  - Object storage (similar to S3)  
  - Stores Iceberg data and metadata  

### 4️\. Physical Storage Layer: Disk/Cloud Storage  
  - The final location where data is stored  
  - MinIO reads and writes data through Volumes


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
`aws.region` is invalid parameter.

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
Move the Doris table to [catalog].[database].[new_table_name] without pre-creating the schema.
```sql
CREATE TABLE iceberg_catalog.database.table AS SELECT * FROM database.table;
```



## References
- [Apache Iceberg](https://iceberg.apache.org/spark-quickstart/#docker-compose)  
- [Apache Doris -  Repository](https://doris.apache.org/docs/3.0/sql-manual/sql-statements/data-modification/backup-and-restore/CREATE-REPOSITORY#examples) 
- [Apache Doris - Iceberg](https://doris.apache.org/docs/lakehouse/lakehouse-best-practices/doris-iceberg)
