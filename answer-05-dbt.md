# Lab 5: Bronze to Silver transformation with dbt, answers

## Questions

### Bronze layer

**The bronze CSV files are left untouched. What is the benefit when a bug is found in a transformation, six months later?**

The raw data is still available exactly as it was received. I can fix the SQL of the model and rebuild the silver layer from the bronze files with `dbt build`, with no data lost. The bronze layer is also a reference to audit and compare the results before and after the fix.

**The source reads the whole file on every query. Which other format and layout would you choose if the orders were ingested every hour for years?**

Parquet files, partitioned by date (for example `year=/month=/day=` in the Hive layout), ideally managed by a table format such as Apache Iceberg or Delta Lake. Parquet is columnar and compressed, and the partitions let the queries and incremental models read only the new or filtered data instead of the whole history.

### Build

**The silver models are views. What happens when a view is queried, and when is it preferable to a table?**

A view stores only the SQL query. When it is queried, DuckDB executes the query again, so it downloads and parses the CSV files from S3 every time. A view is preferable when the data is small, the transformation is cheap, the view is rarely queried, or when the result always needs to be up to date with the source without rebuilding.

**`UUID` values are stored in 16 bytes. How many bytes does the `VARCHAR` representation use?**

36 bytes: 32 hexadecimal characters and 4 hyphens, one byte each, plus the internal overhead of the string. It is more than twice the size of the `UUID` type.

### Data tests

**Which strategy would you choose for a dashboard displaying the age of the customers? For a dashboard displaying the sales per region?**

For the age dashboard, I replace the invalid birthdates with `NULL`, as wrong ages are worse than missing ages, and the users with an unknown age are excluded from the chart. For the sales per region, the birthdate is not used, so I keep all the records and only monitor the issue with a warning. Rejecting the users would remove their orders and the sales would be undercounted.

**The addresses associate states with random zip codes, `KY 01352` for example. Which test would detect it, and which reference data would it require?**

A singular test checking that the zip code of each user is in a valid range for its state. It requires a reference table mapping each state to its zip code ranges, for example built from the USPS or US Census Bureau data, and stored as a dbt seed.

## Exercises

### 1. Add a cleaned column

```sql
lower(trim(username)) as username_normalized
```

The column is documented in `silver.yml` with the `unique` and `not_null` tests.

### 2. Improve order validation

- `quantity` is positive: singular test `tests/assert_orders_quantity_positive.sql`, as no generic test exists for this in dbt Core without a package, plus `not_null`.

```sql
select
    order_id,
    quantity
from {{ ref('stg_orders') }}
where quantity <= 0
```

- `product` belongs to the expected list: generic `accepted_values` test, plus `not_null`.
- every `user_id` exists in `stg_users`: generic `relationships` test, and the singular test `assert_orders_have_users`.

### 3. Clean invalid birthdates

```sql
first_orders as (
    select user_id, min(ordered_at) as first_ordered_at
    from {{ ref('stg_orders') }}
    group by user_id
),

validated as (
    select
        t.*,
        coalesce(f.first_ordered_at >= t.birthdate, true) as birthdate_is_valid
    from typed t
    left join first_orders f on t.user_id = f.user_id
)

select
    ...
    case when birthdate_is_valid then birthdate end as birthdate,
    birthdate_is_valid,
    ...
from validated
```

A user without any order keeps his birthdate. The test `assert_orders_after_user_birth` now only checks the valid birthdates, and the `severity='warn'` configuration is removed:

```sql
where u.birthdate_is_valid
  and o.ordered_at < u.birthdate
```

Before the change, the test failed with 288 rows. It now passes.

### 4. Detect invalid ZIP codes

The seed `seeds/states.csv` lists every state, territory and military region code with the range of the first 3 digits of its zip codes. It also fixes the `relationships` test on `state`, which referenced a `states` model that did not exist. The test `tests/assert_users_zip_matches_state.sql`:

```sql
{{ config(severity='warn') }}

select
    u.user_id,
    u.state,
    u.zip_code
from {{ ref('stg_users') }} u
where not exists (
    select 1
    from {{ ref('states') }} s
    where s.code = u.state
      and cast(left(u.zip_code, 3) as integer) between s.zip_prefix_min and s.zip_prefix_max
)
```

It returns 49 users out of 50, as the generator associates random zip codes with the states.

- **Where does the reference data come from?** From an official source, such as the USPS zip code prefixes or the ZCTA files of the US Census Bureau.
- **Is it stored as a dbt seed?** Yes. It is small, changes rarely, and is versioned in Git with the project. A large or frequently updated dataset would be ingested as a source instead.
- **Does the validation happen in the silver layer?** Yes, silver is the layer where the data is validated. The test only warns, the rows are not deleted, and a flag column could be added if the downstream models need it.
- **What happens when the reference data changes?** The seed is updated, then `dbt seed` and `dbt build` reload it and run the tests again. The results of the test can change for old data too, so the seed changes are tracked in Git, or with a snapshot to keep the history.

### 5. Materialization experiment

- **Where is the data stored?** With a view, the data stays in the CSV files on S3 and DuckDB only stores the query. With a table, the transformed data is stored in the `lab_medallion.duckdb` file.
- **When is the transformation executed?** For a view, at every query. For a table, once, during `dbt build`.
- **What happens when the underlying CSV changes?** The view returns the new data immediately. The table keeps the old data until the next `dbt build`.
- **Which approach requires more storage?** The table, as it stores a copy of the data.
- **Which approach avoids re-reading the source files for every query?** The table.

### Parquet export

`stg_orders` is exported to `s3://user-p-maceviot-ece/silver/dbt/orders.parquet`. The view and the Parquet file both return 2829 rows, and an `EXCEPT` in both directions returns 0 rows, so the content is identical. The view runs the transformation on the CSV at every query (logical materialization), while the Parquet file stores the result once (physical materialization).
