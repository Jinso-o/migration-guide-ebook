Great. I’ll gather a representative example of a common Python pipeline (e.g., using Pandas or requests with the Jaffle Shop dataset or a REST API endpoint), and show how to convert it into a dlt pipeline—focusing on loader replacement, resource wrapping, and incremental configuration. I’ll also find or generate clear visuals and code that align with the voice and structure you’ve used so far.

I’ll be back soon with a complete draft and visuals you can use or refine for the chapter.


# Chapter 3: Converting an Existing Python Pipeline

In this chapter, we'll take a common scenario – you already have a Python data pipeline – and walk through how to migrate it to **dlt**. We will cover the anatomy of a dlt pipeline, and then provide a step-by-step conversion of a simple pipeline (originally using pandas) into a dlt-powered pipeline with incremental loading. The tone and style will follow the same accessible, tutorial-like manner as in Chapter 1.

## Anatomy of a dlt Pipeline

Before converting a pipeline, it's important to understand the key components of dlt: **sources**, **resources**, the **pipeline** itself, and the **destination**. These pieces work together to handle data extraction and loading in a streamlined way.

&#x20;*Diagram: A high-level overview of a dlt pipeline. On the left, various data inputs (JSON records, DataFrames, Python generators, etc., often provided via dlt resources within a source) feed into the dlt pipeline. The pipeline (center) automatically normalizes nested data structures and infers schema, then loads the processed data into the destination on the right (e.g., a database or data warehouse).*

* **Source:** A source in dlt is a logical grouping of one or more data-producing resources (for example, all endpoints of a single API can be grouped into one source). Grouping resources in a source lets you define common configuration (like API credentials or base URLs) only once. In other words, a dlt source represents a data source system (an API, a database, etc.) and contains the resources needed to extract from that system.

* **Resource:** A resource is an individual data extractor – typically a Python function (which can be synchronous or async) that yields data records. You define a resource by decorating a function with `@dlt.resource`. Instead of returning data, a resource function uses Python's `yield` to generate data items (e.g., dictionaries, lists of dicts, or pandas DataFrames). Resources can be configured with options like a custom name (which maps to the destination table name), a primary key, and a write disposition (how to load data, e.g. append vs. merge) to control incremental behavior. The resource is the core unit that actually fetches or produces data.

* **Pipeline:** A dlt pipeline is the orchestrator that *runs* your resources/sources and handles getting data to the destination. When you create a pipeline (using `dlt.pipeline()`), you specify a destination and dataset name. Calling `pipeline.run(...)` on your data (which can be raw Python iterables, a resource, or an entire source) triggers dlt to **extract** the data, **normalize** it (infer schema, handle nested JSON by creating subtables, etc.), and **load** it to the destination automatically. The pipeline takes care of schema discovery and evolution, so you can "just load" the data without writing separate schema or load scripts. Essentially, the pipeline coordinates the end-to-end EL (Extract and Load) process in one call.

* **Destination:** The destination is where your data lands after the pipeline runs – for example, a data warehouse like BigQuery, a relational database, or even a local file-based database. dlt supports many destinations (BigQuery, DuckDB, Snowflake, Redshift, Delta Lake on cloud storage, etc.). You specify the destination when initializing the pipeline, and dlt will handle creating the dataset (schema) and tables if needed, and loading the data into that destination.

With these concepts in mind, let's see how they come together when converting a typical Python pipeline to use dlt.

## Example: Migrating a Python Pipeline to dlt

To make this concrete, consider a simple example pipeline. Say we have a script that retrieves data from an HTTP API and loads it into a database using pandas. We'll use a fictitious "sales" API for this illustration. First, we'll show the original pipeline using standard Python/pandas code, and then we'll demonstrate how to convert it to a dlt pipeline step by step.

### Original Pipeline (using pandas)

Imagine our original pipeline script pulls a list of sales records from an API and writes them to a SQLite database. It might look like this:

```python
import requests
import pandas as pd
import sqlite3

# Connect to SQLite database (local file database)
conn = sqlite3.connect("sales_data.db")

# 1. Extract: Fetch data from an API (e.g., sales records)
response = requests.get("https://api.example.com/sales?limit=100")
response.raise_for_status()
data = response.json()              # assume JSON contains {"sales": [...]}

# 2. Transform: Load JSON data into a pandas DataFrame
sales_records = data["sales"]
df = pd.DataFrame(sales_records)

# 3. Load: Write the DataFrame to a SQL table
df.to_sql("sales", conn, if_exists="replace", index=False)
conn.close()
```

In this script, we manually handle extraction (HTTP GET request), transformation (putting data into a DataFrame), and loading (using `to_sql` to write to the database). While this works, there are a few limitations:

* **Manual schema handling:** We rely on pandas to create the SQL table. If the JSON has nested structures or if schema changes over time, we'd need to adapt the code. There's no built-in schema evolution here.
* **No incremental update:** Each run replaces the entire table (`if_exists="replace"`). If we wanted to update incrementally (only new records), we would have to add logic to fetch only new data or merge with existing data ourselves.
* **No pipeline state:** If the API provided new data periodically, our script doesn't remember what it last loaded. We would need to track the last ID or timestamp processed, perhaps in a file or separate table, to avoid re-loading the same records.

Now, let's convert this pipeline to **dlt**, addressing these points.

### Converting to a dlt Pipeline

We will convert the pipeline in three main steps:

1. **Replace the manual loader with a dlt pipeline.**
2. **Turn the extraction logic into a dlt resource.**
3. **Configure incremental extraction and loading.**

By the end, the data flow and outcome will be the same, but our new pipeline will be easier to maintain and support incremental updates automatically.

#### 1. Replacing the Manual Loader with dlt

In the original script, after fetching and preparing the data, we manually write to the database using `to_sql`. With dlt, we let the pipeline handle loading. We initialize a dlt pipeline and call `run` with our data, and dlt will take care of creating the table (with appropriate schema) and inserting the data.

For example, using DuckDB (a fast local database) as the destination, we could do:

```python
import dlt

# Initialize a dlt pipeline with a destination and dataset name
pipeline = dlt.pipeline(destination="duckdb", dataset_name="sales_data")

# Run the pipeline with data (for now, use the same data we got from the API)
info = pipeline.run(sales_records, table_name="sales")
print(info)
```

In this snippet, `dlt.pipeline(...)` creates a pipeline object configured to load to DuckDB (storing data in a local DuckDB file). The `pipeline.run(sales_records, table_name="sales")` call hands the list of sales records to dlt. Under the hood, dlt will:

* Infer the schema of the data (e.g., columns for each field in the JSON).
* Normalize the data if there are nested structures.
* Load the data into the "sales" table in the destination, creating the table if needed.

All of that happens with one call, thanks to dlt's ability to infer schema and handle the loading process automatically. We no longer need to use pandas for loading, and we didn't have to write any SQL; the pipeline did it for us. The `info` object returned contains metadata about the load (like how many rows were loaded, any package IDs, etc.).

*Note:* You can use any supported destination. Here we showed DuckDB for simplicity, but dlt could just as easily load to BigQuery, PostgreSQL, Snowflake, etc., by changing the `destination` parameter.

#### 2. Turning the Extraction Logic into a dlt Resource

Right now, we have dlt loading a pre-fetched list (`sales_records`). A more **DLT-native** approach is to encapsulate the data extraction in a function and decorate it as a resource. This way, the pipeline can call our function to get the data when running.

Let's convert the API call into a resource function:

```python
import dlt
import requests

@dlt.resource(name="sales", write_disposition="append")
def fetch_sales_data():
    url = "https://api.example.com/sales?limit=100"
    response = requests.get(url)
    response.raise_for_status()
    data = response.json()
    sales_records = data["sales"]
    # Yield each record (as a dict) to dlt
    for record in sales_records:
        yield record

# Create pipeline and run the resource
pipeline = dlt.pipeline(destination="duckdb", dataset_name="sales_data")
pipeline.run(fetch_sales_data())
```

Let's break down what changed here:

* We created a function `fetch_sales_data()` that does the API request and **yields** each sales record one by one. By using `yield` (or `yield from`), we turn this function into a generator. dlt resources must yield data rather than return it, so that dlt can handle data in streams/chunks efficiently.
* We decorated this function with `@dlt.resource`. This tells dlt that `fetch_sales_data` is a resource function. We gave it a `name="sales"` which will be the default name for the destination table, and we set `write_disposition="append"` for now (meaning on each run, new data will be appended to the table, not replacing it).
* In the pipeline.run, instead of passing raw data, we pass `fetch_sales_data()`. Calling the resource function without arguments returns a **resource object** that the pipeline can run. (Alternatively, we could pass the function itself, e.g. `pipeline.run(fetch_sales_data)`, but using `fetch_sales_data()` is more explicit in executing the generator).

With this setup, running the pipeline will trigger our resource to execute the API call and stream data to the pipeline. We still get the same "sales" table in DuckDB, but now the extraction step is built into the dlt workflow. We could add more resources easily if we had other endpoints (for example, a `fetch_customers_data` resource), and group them into a source if needed.

**Why use a source?** If we had multiple related resources (say `fetch_sales_data` and `fetch_customers_data`), we could group them under a single `@dlt.source` (e.g., `@dlt.source def sales_api(): yield fetch_sales_data() and yield fetch_customers_data()`). A source function returns one or more resources (using `yield` to yield each resource). This grouping is useful to share configuration like API tokens or base URLs among resources. In our simple case with one resource, we didn't strictly need a separate source; we can run the resource directly. But it's good to know that sources are available for organizing multiple resources logically.

At this point, our dlt pipeline does the same job as the original script, but with less manual effort in schema management and loading. Next, let's address incremental loading.

#### 3. Configuring Incremental Extract and Load

One of the biggest advantages of using dlt is easier **incremental loading**. Incremental loading means that after the initial load of data, subsequent runs only fetch and load new or updated records, rather than re-processing everything. dlt provides mechanisms to handle this both in extraction and loading:

* **Incremental extraction using state:** dlt pipelines maintain a *state* – basically a dictionary that persists across runs. We can use this state to remember things like "the last record ID we processed" or "the timestamp of the last fetch". By storing a cursor in state, our resource can request only new data from the API next time. For example, if the API supports a parameter like `since_id` or `updated_after`, we can keep track of the last ID/timestamp and use it on the next run.
* **Incremental loading using write dispositions:** dlt can deduplicate or upsert data on load by using appropriate `write_disposition` settings on resources. We already used `append` for a simple add-only load. If we want to avoid duplicates or update existing records, we can use `write_disposition="merge"` with a primary key. This way, dlt will merge new data with old data on that key, ensuring each record appears only once. For example, if a sales record with a certain ID already exists, a merge will update it (or skip inserting a duplicate) rather than insert another copy.

Let's update our resource to load incrementally. Suppose the API allows fetching new sales since a given ID. We will do two things: (a) use the pipeline state to store the last seen `sale_id`, and (b) switch to `write_disposition="merge"` with the primary key set to that ID, so dlt can upsert.

```python
@dlt.resource(name="sales", primary_key="id", write_disposition="merge")
def fetch_sales_data_incremental():
    # Get the last processed ID from state (or None if first run)
    last_id = dlt.current.resource_state().setdefault("last_id", None)
    url = "https://api.example.com/sales"
    params = {}
    if last_id:
        params["since_id"] = last_id  # ask API for records with ID greater than last_id
    response = requests.get(url, params=params)
    response.raise_for_status()
    data = response.json()
    sales_records = data["sales"]
    # Yield each new record
    for record in sales_records:
        yield record
    # After yielding all records, update the state with the max ID seen
    if sales_records:
        max_id = max(item["id"] for item in sales_records)
        dlt.current.resource_state()["last_id"] = max_id
```

A few important points in this incremental version:

* We changed the decorator to `@dlt.resource(primary_key="id", write_disposition="merge")`. Now dlt knows that the `"id"` field is the unique primary key for our records, and we want to merge updates. This means on the destination side, if we run this pipeline multiple times, each sales record will only appear once (the record with the same `id` will be updated rather than duplicated).
* We use `dlt.current.resource_state()` to access a state dictionary specific to this resource. We call `.setdefault("last_id", None)` to retrieve the last saved ID, or initialize it to `None` if this is the first run. The state is stored alongside the data in the destination, so it persists between runs.
* We include a `since_id` parameter in the API request if `last_id` was found. This instructs the API to return only records newer than the last seen ID (assuming the API supports such a parameter).
* We yield all the records as before. After yielding, we update the state: we take the maximum `id` from the fetched records and save it as the new `"last_id"` in state for next time.
* On the next pipeline run, `last_id` will be populated from the previous run's state, so the function will fetch only records with a higher ID, yield those, and update the state again. This way, each run processes a new batch of data without repeating the old ones.

dlt's state management makes this pattern straightforward – we didn't have to write to an external file or table to keep track of the last ID; it's handled as part of the pipeline state. The state update is committed atomically with the data load, meaning if the pipeline run succeeds, the state is saved (and if it fails, the state doesn’t advance, avoiding gaps).

Additionally, using `write_disposition="merge"` with a primary key ensures even if an old record appears again or if we accidentally fetch overlapping data, dlt will merge it to avoid duplicate entries. (If the API instead provided an `updated_at` timestamp for incremental loading, we could use a similar approach with a timestamp cursor. In fact, dlt supports a declarative incremental option where you can specify a cursor field in the resource configuration. For instance, one could configure a resource to use an `"updated_at"` field as an incremental cursor, so that dlt automatically uses the last seen timestamp for the next fetch – but for clarity we implemented the logic manually here.)

Finally, let's run the pipeline with our incremental resource:

```python
pipeline = dlt.pipeline(destination="duckdb", dataset_name="sales_data")
info = pipeline.run(fetch_sales_data_incremental())
print(info)
```

On the first run, it will load all sales (up to the limit, or all available data if no limit). On subsequent runs, it will fetch only new sales added since the last run, appending or merging them into the same table. The result is an up-to-date **`sales`** table that grows with new data, without duplicates and without re-fetching the entire dataset each time.

### Benefits of the dlt Pipeline Approach

By converting our pipeline to dlt, we gained several benefits:

* **Less boilerplate:** We no longer manually handle database connections or write DataFrame to SQL. dlt took care of connecting to the destination and loading data in optimal batches, with schema inference and creation done automatically.
* **Incremental logic built-in:** With a few lines, we added robust incremental loading. dlt's pipeline state and merge functionality save us from writing our own checkpointing or upsert SQL.
* **Scalability:** The resource function yields records, which means dlt can handle large datasets by streaming/chunking data. We could easily extend the resource to fetch data page by page (yielding one page at a time) and dlt would accumulate those. This is more memory-efficient than loading everything into a single DataFrame in memory. dlt is designed to handle iterators and even parallel extraction for scalability.
* **Maintainability:** Configuration like API URLs, parameters, and credentials can be cleanly managed (e.g., using `dlt.secrets` for sensitive info, and grouping resources in sources). The pipeline code is declarative about what to extract and how to load, rather than mixing extraction with loading details.
* **Destination flexibility:** Today we used DuckDB, but tomorrow we could switch `destination="bigquery"` or another destination with minimal changes. dlt pipelines abstract away the specifics of the target data store.

In summary, migrating an existing Python/pandas pipeline to dlt involves wrapping your data extraction logic into dlt resources/sources and letting the dlt pipeline handle the heavy lifting of loading (and optionally incremental state tracking). The result is a pipeline that's easier to scale, incrementally update, and adapt over time.
