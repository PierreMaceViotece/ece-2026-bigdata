# Lab 4: SQL analytics with DuckDB, answers

## Exercises

### 1. Average number of orders per user and average quantity per order

```sql
SELECT
  round(count(*) / count(DISTINCT user_uuid), 2) AS avg_orders_per_user,
  round(avg(quantity), 2) AS avg_quantity_per_order
FROM orders;
```

### 2. First and last order of each user and number of days between them

```sql
SELECT u.username,
  min(o.date) AS first_order,
  max(o.date) AS last_order,
  date_diff('day', min(o.date), max(o.date)) AS days_between
FROM orders o JOIN users u ON o.user_uuid = u.uuid
GROUP BY ALL
ORDER BY days_between DESC;
```

### 3. Hour of the day with the highest quantity sold

```sql
SELECT hour(date) AS hour, sum(quantity) AS quantity
FROM orders
GROUP BY hour
ORDER BY quantity DESC
LIMIT 1;
```

### 4. Month-over-month variation of the quantity sold per product

```sql
WITH monthly AS (
  SELECT strftime(date, '%Y-%m') AS month, product, sum(quantity) AS quantity
  FROM orders
  GROUP BY month, product
)
SELECT month, product, quantity,
  lag(quantity) OVER w AS prev_quantity,
  round(100 * (quantity - lag(quantity) OVER w) / lag(quantity) OVER w, 1) AS variation_pct
FROM monthly
WINDOW w AS (PARTITION BY product ORDER BY month)
ORDER BY product, month;
```

### 5. Users who ordered every product at least once

```sql
SELECT u.username, u.name
FROM orders o JOIN users u ON o.user_uuid = u.uuid
GROUP BY ALL
HAVING count(DISTINCT o.product) = (SELECT count(DISTINCT product) FROM orders);
```

## Questions

### S3 configuration

**The persistent secret is stored in a file of your home directory. What are the risks, and why are they limited on Onyxia?**

The keys are stored in clear in a file, so any process or person with access to the home directory can read them, and the file can leak through a backup or a careless `git add`. On Onyxia the risk is limited because the credentials are temporary and expire after a few hours, and each service runs in its own pod that belongs to one user only.

**The secret is visible to any process running as your user. How would you restrict the access to S3 for a Kubernetes Job?**

I would give the Job its own credentials with minimal rights, for example read only on one bucket or one prefix. They are stored in a Kubernetes Secret mounted only in the pod of the Job. Even better, the Job gets its own identity (a service account mapped to a role) and receives short-lived tokens instead of static keys.

### Query the bronze layer

**Why does `approx_unique` return 40 and 3167 instead of 50 and 2829?**

`approx_unique` uses an approximate algorithm (HyperLogLog). It is fast and uses little memory on big datasets, but it returns an estimate with an error margin, not the exact count. `count(DISTINCT ...)` gives the exact value.

**Which issue may occur with a large file whose first rows are not representative?**

DuckDB infers the types from a sample of the first rows. If these rows are not representative, a column gets the wrong type, for example BIGINT while some later rows contain text. The read then fails with a conversion error, or the values are badly converted. The fix is a bigger `sample_size` (`-1` reads the whole file) or giving the column types explicitly.

### Parquet export

**Why is the `uuid` column barely compressed?**

Each `uuid` is random and unique, so there is no repetition to exploit. Dictionary encoding is useless and Snappy finds no pattern to compress.

**Why is the compressed size of some columns larger than their uncompressed size?**

These columns (binary timestamps, small integers) are already compact. Compression gains nothing on them and adds a small overhead (headers, block metadata), so the result is slightly bigger.

### Hive partitioning

**Why is partitioning by `uuid` a bad idea?**

Every order has a different `uuid`, so it creates one directory and one tiny file per order, which means millions of small files. On object storage each request has a high latency and listing many objects is slow and costly, so reading the dataset becomes very slow (small files problem).

**Which partition column would you choose for a dataset of orders growing every day?**

The order date, at the day or month level (for example `year/month/day`). New data is written in new partitions without touching the old ones, and most queries filter on a time range, so they read only the needed partitions.

### CSV vs. Parquet at scale

**How many row groups does the file contain, and how many are skipped by the filter `date >= '2100-01-01'`?**

My file contains 3 row groups (123,573, 124,276 and 4,567 rows). Their dates go from 2020-01-01 to 2048-10-17, so the maximum date of every row group is before 2100. DuckDB compares the filter with the min and max statistics of the footer and skips all 3 row groups. This is visible in the measure: the filtered query receives only 16.0 KiB with 1 GET request, which is the footer, and no data at all.

**The orders are sorted by date. What would happen to the efficiency of the filter if they were shuffled?**

Each row group would contain dates from the whole range, so its min and max would cover the first to the last date. No row group could be skipped and DuckDB would read the whole `date` column.

**Compare the execution time of the CSV and Parquet queries. Which part of the difference is due to the network, which part to the parsing of the CSV file?**

The CSV query took 1.31 s and the Parquet query 0.377 s, so the CSV query is about 3.5 times slower. For the network part, the CSV query downloads the whole file (28.2 MiB) while the Parquet query downloads only the footer and the `product` and `quantity` column chunks (203.5 KiB), about 140 times less data. For the parsing part, the CSV file is text that DuckDB parses entirely, all columns, while Parquet stores binary and compressed values that are decoded for 2 columns only. To separate both parts, I can download the CSV file locally and run the same query on the local file: the remaining time is the parsing, and the difference with the S3 query is the network.
