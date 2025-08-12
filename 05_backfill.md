<p align="center">
“Replace the ship, keep the voyage.” - The Story of the Ship Of Theseus
</p>

# Loading historical data when migrating to `dlt`


When creating a new pipeline for a source that has been running for a while or when migrating an existing pipeline to `dlt`, a crucial step is to **bring over years of historical data** into your new setup. We recommend two primary strategies for this:

* **Stitching** – keep historical data in place and present it alongside new data via a virtual union (e.g. a SQL view).
* **Backfilling** – re-import *all* historical data into your new destination.

## Option 1: 🪡 Stitching Old and New Data

If your legacy data is still accessible in its original database or warehouse, and you prefer not to reload it immediately, you can **stitch it together** with new `dlt`-loaded data using a read-time combination (for example, a **view** that unions the datasets). This essentially creates a *logical combination of multiple tables exposed as a single dataset*.

&#x20;**Figure:** Combining legacy and new data tables using a SQL `UNION ALL` view for a unified dataset. In this example, a view `unified_events` pulls from both the legacy `events` table and the new `dlt`-loaded `events` table, tagging each row with its source (legacy vs. dlt). This approach avoids duplicating the old data in the new system during the transition.

For example, you could create a SQL view that unions the two tables:

```sql
CREATE OR REPLACE VIEW unified_events AS
SELECT *, 'legacy' AS source_flag 
FROM legacy_schema.events
UNION ALL
SELECT *, 'dlt' AS source_flag 
FROM dlt_schema.events;
```

**🟢 When to use stitching**: This strategy is best when you want a fast, low-effort migration with zero data duplication. It assumes your legacy data is static (no longer changing) and lets you **gradually deprecate** the old system over time while new data flows into `dlt`.

* *No immediate reload needed*: You can query all data through one view without moving old records.
* *Legacy data is read-only*: Suitable if historical data won’t be updated further.
* *Phased migration*: Allows verifying the new pipeline in parallel and switching fully to `dlt` at your own pace.

## Option 2: 🕳️ Backfilling Historical Data into `dlt`

If you prefer to have **all data in the new pipeline and destination** meaning eventually retiring the old storage, then you’ll want to perform a **backfill** – i.e. load the historical records into `dlt`. There are a few approaches to backfilling, depending on your data volume and source capabilities:

### A. Full Table Load

This simplest method is to do a one-time **full export** of the entire source table into `dlt`. Use this when the table is reasonably small or the source->destination connection is very fast (e.g. copying from Postgres to BigQuery in one go). This approach runs one complete scan of the table:

```python
from dlt.sources.sql_database import sql_database

source = sql_database("postgresql://user:pass@host:port/db")
resource = source.table("events")   # select the entire 'events' table
pipeline.run(resource)
```

This will load all rows from the legacy `events` table into the pipeline in one run.

* ✅ Simple – the pipeline logic is straightforward (just one big load).
* ❌ Doesn’t scale well for very large datasets – a massive one-shot transfer could time out or exhaust memory.

### B. Chunked Loading (Batching the Backfill)

For large tables (millions of rows or more) **without an easy timestamp/ID cursor**, a good pattern is to break the backfill into **chunks** of rows. Many SQL-based sources support grabbing a subset of rows at a time. In `dlt`, you can specify a `chunk_size` on the resource to limit how many rows to fetch per run:

```python
resource = source.table("events", chunk_size=10_000)
pipeline.run(resource)
```

Each run with `chunk_size` will fetch the next batch of rows (e.g. 10,000 at a time) from the source. You would repeat or automate these pipeline runs to cover the whole dataset – for example via a loop, a script, or an orchestrator (Airflow, etc.) until all chunks are loaded. This **chunked backfill** avoids doing one enormous transfer, reducing the risk of failures and memory issues.

* ✅ Avoids memory/time issues that a single huge load might cause.
* ✅ Backend-agnostic – works even if the source has no natural time or ID field for incremental loading.
* ❌ Requires multiple runs (manual repetition or orchestration) to cover all chunks.

> *Tip:* You can monitor progress by checking how many rows have loaded after each chunk, and stop once it matches the legacy table’s row count. Some pipelines use a loop with a condition or the `dlt` **dataset accessor** to verify when to stop.

### C. Time-Sliced Backfilling (Parallel-Friendly)

If your source data has a **time-based or incrementing ID column**, you can leverage it to backfill in defined slices (e.g. by date range or ID range). In this approach, you parameterize your extraction by a start/end and run many small extractions that cover different portions of history. For example, using `dlt`’s incremental helpers:

```python
@dlt.resource(primary_key="id")
def events(
    updated_at = dlt.sources.incremental(
        "updated_at", 
        initial_value="2020-01-01",   # start of backfill window
        end_value="2023-12-31"        # end of backfill window
    )
):
    # fetch_events is a function that pulls records in the given date range
    yield fetch_events(updated_at.start, updated_at.end)
```

In this example, the resource is defined to load `events` in a date range (from Jan 1, 2020 up to Dec 31, 2023). The `dlt.sources.incremental` helper here defines a cursor on the `updated_at` field with a fixed start and end – essentially splitting the full history into **time windows**. Each invocation (or each parallel task) of this resource could load one slice (say one day, one week, etc. depending on how `fetch_events` is implemented).

Because each slice is independent and the state is defined by the parameters (start/end), this method is **stateless across runs** – you can launch multiple slices in parallel without them interfering. Using an orchestrator (like an Airflow DAG, AWS Lambda jobs, Cloud Workflows, Prefect, Dagster, etc.), you could kick off many slice loads concurrently to drastically speed up the backfill.

&#x20;**Figure:** Serial vs. parallel backfill. The top bar illustrates a sequential backfill of \~1500 days of data (which might take 1500 hours if one day takes 1 hour to load). The stacked bars below illustrate breaking the same range into many small slices (e.g. 500 parallel tasks, each loading \~3 days) – in theory completing in the time of a single slice (\~3 hours), assuming sufficient resources for parallel execution.

In practice, if one day’s data takes \~1 hour to load, doing 1500 days serially would take \~1500 hours. But if you launch 500 parallel loaders, each handling a 3-day window, the total wall-clock time might drop to only \~3 hours. This illustrates the massive speed-up from slicing the backfill and running slices concurrently.

✅ Ideal for very long history loads – you can break years of data into manageable chunks.
✅ Highly parallelizable – slices can be processed concurrently, greatly reducing overall time.
❌ Requires orchestration tooling – you need a way to kick off and coordinate parallel slice jobs (this could be a custom script or a workflow platform, as mentioned).

### ⚡ Pro Tip: Combine Stitching *and* Backfill

These strategies aren’t mutually exclusive – you can use stitching as a quick stop-gap while a backfill runs in the background:

1. **Start with a union view** (stitching) that combines the legacy and new `dlt` data. This gives you seamless access to all data immediately.
2. **Gradually backfill** the historical data into the `dlt` pipeline (using one of the methods above) over time, loading old records into the new destination.
3. Once the backfill catches up and all legacy data now lives in `dlt`, **switch the view** (or your application queries) to reference only the new `dlt` table. You can then retire the old system.

This phased approach minimizes downtime and complexity: users and reports can continue querying a unified view throughout the migration, and you ensure nothing is missed before fully switching to `dlt`.


### Definition of Done (tick as you go)

[Example code](https://colab.research.google.com/drive/1ChFgNy6r_EUmobJslK3q5w3iCBsyEqfP?usp=sharing)

 * Stitching works before backfill → dw.all_orders shows one row: ('legacy', N). (Uses a union view — easy to grasp in a README.) 
GitHub Docs

 * Monthly backfill loads → dw.all_orders shows two sources: legacy + dlt.

 * Re-run a month = no duplicates (MERGE on id).

 * Cutover complete → dw.all_orders shows only ('dlt', N)

> **Outcome:** you’ll leave with a hands-on feel for how stitching gives zero-downtime visibility, while dlt’s stateful, cursor-aware backfill lets you migrate years of data safely and repeatably.
