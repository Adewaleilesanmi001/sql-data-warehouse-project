# SQL Data Warehouse: Medallion Architecture in PostgreSQL

A layered data warehouse built entirely in the database engine: bronze, silver, and gold schemas ingesting CRM and ERP source system data, orchestrated by a PL/pgSQL stored procedure with idempotent loads, audit logging, and exception handling.

## Why this project

Many production data estates, especially in the public sector, are SQL-first: no Spark cluster, no Python runtime near the database. This project demonstrates that the same engineering standards applied in code-based pipelines (idempotency, observability, error handling, layered refinement) can be delivered natively in PostgreSQL.

## Architecture

Bronze (raw landing) -> Silver (cleaned, standardised) -> Gold (business-ready, dimensional)

- **Bronze** holds typed landing tables mirroring two source systems:
  - CRM: `crm_cust_info` (customer identity and demographics), `crm_prd_info` (product master), `crm_sales_details` (sales transactions)
  - ERP: `erp_cust_az12` (customer demographics), `erp_loc_a101` (customer locations), `erp_px_cat_g1v2` (product category reference)
- **Silver** applies cleaning, standardisation, and type correction.
- **Gold** models the data dimensionally for analytics consumption.

## The load procedure

`bronze.load_bronze()` orchestrates ingestion as a single auditable call. For each table it:

1. Announces the step with `RAISE NOTICE` (a narrated, greppable load log)
2. `TRUNCATE`s the target so every run is idempotent: rerunning never duplicates
3. Bulk-loads via `COPY ... WITH (FORMAT CSV, HEADER TRUE)` for speed
4. Captures timing with `clock_timestamp()`
5. Catches failures in an `EXCEPTION WHEN OTHERS` block reporting `SQLERRM`, so the load either completes with a narrated log or fails with a captured reason

```sql
CALL bronze.load_bronze();
```

## Engineering decisions

- **TRUNCATE + COPY over DELETE + INSERT:** faster, and guarantees clean idempotent reloads for a snapshot-style landing zone.
- **RAISE NOTICE narration:** operability. When a load misbehaves at 2am, the log tells you which table and which step.
- **Explicit typed DDL with post-load type correction** (e.g. `ALTER TABLE ... USING ... ::INT`): landing zones receive imperfect types; fixing them is part of the job, not an embarrassment.
- **Schema-per-layer:** bronze/silver/gold as PostgreSQL schemas keeps refinement stages physically separated and permission-controllable.

## Running it

1. PostgreSQL 14+ with a `DataWearhouse` database
2. Run `scripts.sql` to create schemas, tables, and the procedure
3. Update the `COPY` source paths to your local CSV locations
4. `CALL bronze.load_bronze();`

## Roadmap

- Row-count reconciliation checks after each COPY
- Silver-layer cleaning procedures (deduplication, standardisation, referential validation)
- Gold-layer dimensional views (customer and product dimensions, sales fact)
- Migration of file paths to server-side configuration

## Author

Adewale Ilesanmi | [LinkedIn](https://www.linkedin.com/in/adewale-ilesanmi001) | Part of a public portfolio of tested, documented data engineering projects.
