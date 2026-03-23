---
name: polars-data
description: High-performance DataFrame operations with Polars — lazy evaluation, streaming, CSV/Parquet I/O, expressions API
license: MIT
triggers:
  - polars dataframe
  - polars lazy
  - scan csv polars
  - read parquet polars
  - polars groupby
  - polars expressions
  - streaming large dataset
  - polars vs pandas
  - polars pivot
  - polars join
metadata:
  skill-author: gonzih
  data-sources:
    - local files — CSV, Parquet, JSON, NDJSON, IPC/Arrow, AVRO
  byok:
    - none
  compatibility: claude-code>=1.0
---

# Polars Data Processing Skill

## What it does
Uses Polars for high-performance DataFrame operations in Python and Rust. Covers lazy evaluation (scan_csv, scan_parquet) for files larger than memory, the expressions API for complex transformations, groupby/join/pivot/explode operations, streaming execution, writing back to Parquet, and key differences from pandas.

## How to invoke
- "Read this CSV with Polars and compute groupby statistics"
- "Process a 50GB Parquet file without loading into memory"
- "Join two DataFrames on multiple keys with Polars"
- "Pivot this table from long to wide format"
- "Write results to Parquet with Zstandard compression"
- "Compute rolling mean and lag features for time series"

## Key parameters
- `source` — file path or glob pattern (`data/*.parquet`)
- `lazy` — use `scan_*` for lazy/streaming, `read_*` for eager
- `streaming` — `collect(streaming=True)` for out-of-memory execution
- `schema` — explicit dtype mapping `{"col": pl.Int64}`
- `n_rows` — limit rows read (useful for inspection)

## Why Polars over pandas
- 5–100x faster for large datasets (parallelized, SIMD-optimized)
- Lazy execution: plan is optimized before running
- True multi-threading (no GIL issues)
- Streaming: process files larger than RAM
- Expression API: composable, no chained method syntax needed
- Consistent API: no index confusion

## Workflow steps
1. Install: `pip install polars`
2. Use `pl.scan_*()` for lazy evaluation (preferred for large files)
3. Chain `.filter()`, `.select()`, `.with_columns()`, `.group_by()` etc.
4. Call `.collect()` or `.collect(streaming=True)` to execute
5. Write results with `.write_parquet()`, `.write_csv()`

## Working code example

```python
import polars as pl
from pathlib import Path

# ── Reading data ──────────────────────────────────────────────────────────────

def read_csv_eager(filepath: str, n_rows: int = None) -> pl.DataFrame:
    """Read CSV eagerly (fully into memory)."""
    return pl.read_csv(
        filepath,
        n_rows=n_rows,
        infer_schema_length=1000,
        try_parse_dates=True,
        null_values=["", "NA", "N/A", "null"],
    )


def scan_csv_lazy(filepath: str) -> pl.LazyFrame:
    """Lazily scan a CSV — no data loaded yet."""
    return pl.scan_csv(
        filepath,
        infer_schema_length=1000,
        try_parse_dates=True,
        null_values=["", "NA", "N/A", "null"],
    )


def scan_parquet_lazy(path: str) -> pl.LazyFrame:
    """Lazily scan a Parquet file or glob pattern."""
    return pl.scan_parquet(path, low_memory=False)


# ── Expressions API ───────────────────────────────────────────────────────────

def demonstrate_expressions(df: pl.DataFrame) -> pl.DataFrame:
    """
    Show the Polars expressions API — composable column operations.
    Assumes df has columns: date, category, value, quantity
    """
    return (
        df.with_columns([
            # String ops
            pl.col("category").str.to_uppercase().alias("category_upper"),
            pl.col("category").str.len_chars().alias("category_len"),

            # Numeric ops
            (pl.col("value") * pl.col("quantity")).alias("revenue"),
            pl.col("value").log(base=10.0).alias("log_value"),

            # Date ops (if date column exists)
            pl.col("date").dt.year().alias("year"),
            pl.col("date").dt.month().alias("month"),
            pl.col("date").dt.weekday().alias("weekday"),  # 0=Mon

            # Conditionals
            pl.when(pl.col("value") > pl.col("value").mean())
              .then(pl.lit("above_avg"))
              .otherwise(pl.lit("below_avg"))
              .alias("relative_value"),

            # Window functions (like SQL OVER)
            pl.col("value").rank("ordinal").over("category").alias("rank_within_category"),
            pl.col("value").cum_sum().over("category").alias("running_total"),
        ])
    )


# ── GroupBy / aggregation ─────────────────────────────────────────────────────

def groupby_stats(df: pl.DataFrame, group_cols: list[str], value_col: str) -> pl.DataFrame:
    """Comprehensive groupby statistics."""
    return (
        df.group_by(group_cols)
        .agg([
            pl.col(value_col).count().alias("count"),
            pl.col(value_col).sum().alias("sum"),
            pl.col(value_col).mean().alias("mean"),
            pl.col(value_col).median().alias("median"),
            pl.col(value_col).std().alias("std"),
            pl.col(value_col).min().alias("min"),
            pl.col(value_col).max().alias("max"),
            pl.col(value_col).quantile(0.25).alias("p25"),
            pl.col(value_col).quantile(0.75).alias("p75"),
            pl.col(value_col).n_unique().alias("n_unique"),
            pl.col(value_col).null_count().alias("null_count"),
        ])
        .sort(group_cols)
    )


# ── Joins ─────────────────────────────────────────────────────────────────────

def join_examples(left: pl.DataFrame, right: pl.DataFrame) -> dict[str, pl.DataFrame]:
    """Demonstrate different join types."""
    return {
        "inner": left.join(right, on="id", how="inner"),
        "left": left.join(right, on="id", how="left"),
        "full": left.join(right, on="id", how="full"),
        "anti": left.join(right, on="id", how="anti"),  # rows in left NOT in right
        # Multi-key join
        "multi_key": left.join(right, on=["id", "date"], how="inner"),
        # Join with different column names
        "cross_key": left.join(right, left_on="left_id", right_on="right_id", how="inner"),
    }


# ── Pivot / melt ──────────────────────────────────────────────────────────────

def pivot_wide(df: pl.DataFrame) -> pl.DataFrame:
    """Pivot from long to wide format."""
    return df.pivot(
        values="value",
        index="date",
        on="category",
        aggregate_function="sum",  # or 'mean', 'first', 'last', 'min', 'max'
    )


def melt_long(df: pl.DataFrame, id_cols: list[str], value_cols: list[str]) -> pl.DataFrame:
    """Melt from wide to long format (like pandas.melt)."""
    return df.unpivot(
        on=value_cols,
        index=id_cols,
        variable_name="variable",
        value_name="value",
    )


# ── Time series features ──────────────────────────────────────────────────────

def time_series_features(df: pl.DataFrame, date_col: str, value_col: str) -> pl.DataFrame:
    """Add rolling window and lag features for time series."""
    return (
        df.sort(date_col)
        .with_columns([
            pl.col(value_col).shift(1).alias(f"{value_col}_lag1"),
            pl.col(value_col).shift(7).alias(f"{value_col}_lag7"),
            pl.col(value_col).rolling_mean(window_size=7).alias(f"{value_col}_ma7"),
            pl.col(value_col).rolling_mean(window_size=30).alias(f"{value_col}_ma30"),
            pl.col(value_col).rolling_std(window_size=7).alias(f"{value_col}_std7"),
            pl.col(value_col).pct_change().alias(f"{value_col}_pct_change"),
            # Difference
            pl.col(value_col).diff(1).alias(f"{value_col}_diff1"),
        ])
    )


# ── Lazy pipeline with streaming ──────────────────────────────────────────────

def process_large_file(input_path: str, output_path: str, min_value: float = 0.0) -> int:
    """
    Process a large CSV/Parquet file with lazy evaluation + streaming.
    Handles files larger than RAM. Returns number of rows written.
    """
    lf = pl.scan_parquet(input_path) if input_path.endswith(".parquet") else pl.scan_csv(input_path)

    result = (
        lf
        .filter(pl.col("value") >= min_value)
        .with_columns([
            (pl.col("value") * 1.1).alias("value_adjusted"),
            pl.col("category").str.to_uppercase().alias("category"),
        ])
        .group_by(["category", pl.col("date").dt.year().alias("year")])
        .agg([
            pl.col("value_adjusted").sum().alias("total"),
            pl.col("value_adjusted").count().alias("count"),
        ])
        .sort(["year", "category"])
    )

    # collect(streaming=True) processes in chunks — memory-safe for huge files
    df = result.collect(streaming=True)
    df.write_parquet(output_path, compression="zstd", compression_level=3)
    return len(df)


# ── Writing data ──────────────────────────────────────────────────────────────

def write_parquet(df: pl.DataFrame, filepath: str, compression: str = "zstd") -> None:
    """Write DataFrame to Parquet (recommended for large datasets)."""
    df.write_parquet(
        filepath,
        compression=compression,  # 'snappy', 'gzip', 'brotli', 'lz4', 'zstd', 'uncompressed'
        statistics=True,
        row_group_size=100_000,
    )


def write_csv(df: pl.DataFrame, filepath: str) -> None:
    """Write DataFrame to CSV."""
    df.write_csv(filepath, separator=",", null_value="")


# ── Pandas comparison cheatsheet ──────────────────────────────────────────────
PANDAS_TO_POLARS = """
pandas                          polars
-------                         ------
df[df['col'] > 5]               df.filter(pl.col('col') > 5)
df['col'] * 2                   pl.col('col') * 2  (in with_columns)
df.groupby('a').sum()           df.group_by('a').agg(pl.all().sum())
df.merge(other, on='id')        df.join(other, on='id', how='inner')
df.pivot_table(...)             df.pivot(values=..., index=..., on=...)
df.melt(...)                    df.unpivot(on=..., index=...)
df['col'].rolling(7).mean()     pl.col('col').rolling_mean(7)
df.sort_values('col')           df.sort('col')
df.drop_duplicates()            df.unique()
df.fillna(0)                    df.fill_null(0)
df.apply(func, axis=1)          df.map_rows(func)  — or use expressions
"""


# --- Example usage ---
if __name__ == "__main__":
    import io

    # Create sample data
    csv_data = """date,category,value,quantity
2024-01-01,A,100.5,10
2024-01-02,B,200.0,5
2024-01-03,A,150.0,8
2024-01-04,B,300.0,12
2024-01-05,A,120.0,7
2024-01-06,B,250.0,9
"""
    df = pl.read_csv(io.StringIO(csv_data), try_parse_dates=True)
    print("Schema:", df.schema)
    print(df)

    # GroupBy
    stats = groupby_stats(df, ["category"], "value")
    print("\nGroupBy stats:")
    print(stats)

    # Lazy pipeline
    result = (
        pl.scan_csv(io.StringIO(csv_data), try_parse_dates=True)
        .filter(pl.col("value") > 150)
        .with_columns((pl.col("value") * pl.col("quantity")).alias("revenue"))
        .group_by("category")
        .agg(pl.col("revenue").sum())
        .collect()
    )
    print("\nRevenue by category (value > 150):")
    print(result)

    # Time series features
    ts = time_series_features(df, "date", "value")
    print("\nTime series features:")
    print(ts.select(["date", "value", "value_ma7", "value_pct_change"]))
```

## Live Data Sources
- **Polars documentation**: https://docs.pola.rs/
- **Polars GitHub**: https://github.com/pola-rs/polars
- **User guide**: https://docs.pola.rs/user-guide/
- **Expressions API**: https://docs.pola.rs/api/python/stable/reference/expressions/
- **Lazy evaluation**: https://docs.pola.rs/user-guide/lazy/
