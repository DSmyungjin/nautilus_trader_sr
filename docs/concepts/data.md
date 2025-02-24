# Data

The NautilusTrader platform defines a range of built-in data types crafted specifically to represent 
a trading domain:

- `OrderBookDelta` (L1/L2/L3) - Most granular order book updates
- `OrderBookDeltas` (L1/L2/L3) - Bundles multiple order book deltas
- `QuoteTick` - Top-of-book best bid and ask prices and sizes
- `TradeTick` - A single trade/match event between counterparties
- `Bar` - OHLCV 'bar' data, aggregated using a specific *method*
- `Ticker` - General base class for a symbol ticker
- `Instrument` - General base class for a tradable instrument
- `VenueStatus` - A venue level status event
- `InstrumentStatus` - An instrument level status event
- `InstrumentClose` - An instrument closing price

Each of these data types inherits from `Data`, which defines two fields:
- `ts_event` - The UNIX timestamp (nanoseconds) when the data event occurred
- `ts_init` - The UNIX timestamp (nanoseconds) when the object was initialized

This inheritance ensures chronological data ordering (vital for backtesting), while also enhancing analytics.

Consistency is key; data flows through the platform in exactly the same way for all system contexts (`backtest`, `sandbox`, `live`)
primarily through the `MessageBus` to the `DataEngine` and onto subscribed or registered handlers.

For those seeking customization, the platform supports user-defined data types. Refer to the advanced [Custom/Generic data guide](advanced/custom_data.md) for more details.

## Loading data

NautilusTrader facilitates data loading and conversion for three main use cases:
- Populating the `BacktestEngine` directly to run backtests
- Persisting the Nautilus-specific Parquet format for the data catalog via `ParquetDataCatalog.write_data(...)` to be later used with a `BacktestNode`
- For research purposes (to ensure data is consistent between research and backtesting)

Regardless of the destination, the process remains the same: converting diverse external data formats into Nautilus data structures.

To achieve this, two main components are necessary:
- A type of DataLoader (normally specific per raw source/format) which can read the data and return a `pd.DataFrame` with the correct schema for the desired Nautilus object
- A type of DataWrangler (specific per data type) which takes this `pd.DataFrame` and returns a `list[Data]` of Nautilus objects

### Data loaders

Data loader components are typically specific for the raw source/format and per integration. For instance, Binance order book data is stored in its raw CSV file form with
an entirely different format to [Databento Binary Encoding (DBN)](https://docs.databento.com/knowledge-base/new-users/dbn-encoding/getting-started-with-dbn) files.

### Data wranglers

Data wranglers are implemented per specific Nautilus data type, and can be found in the `nautilus_trader.persistence.wranglers` module.
Currently there exists:
- `OrderBookDeltaDataWrangler`
- `QuoteTickDataWrangler`
- `TradeTickDataWrangler`
- `BarDataWrangler`

```{warning}
At the risk of causing confusion, there are also a growing number of DataWrangler v2 components, which will take a `pd.DataFrame` typically
with a different fixed width Nautilus arrow v2 schema, and output pyo3 Nautilus objects which are only compatible with the new version
of the Nautilus core, currently in development.

**These pyo3 provided data objects are not compatible where the legacy Cython objects are currently used (adding directly to a `BacktestEngine` etc).**
```

### Transformation pipeline

**Process flow:**
1. Raw data (e.g., CSV) is input into the pipeline
2. DataLoader processes the raw data and converts it into a `pd.DataFrame`
3. DataWrangler further processes the `pd.DataFrame` to generate a list of Nautilus objects
4. The Nautilus `list[Data]` is the output of the data loading process

```
  ┌──────────┐    ┌──────────────────────┐                  ┌──────────────────────┐
  │          │    │                      │                  │                      │
  │          │    │                      │                  │                      │
  │ Raw data │    │                      │  `pd.DataFrame`  │                      │
  │ (CSV)    ├───►│      DataLoader      ├─────────────────►│     DataWrangler     ├───► Nautilus `list[Data]`
  │          │    │                      │                  │                      │
  │          │    │                      │                  │                      │
  │          │    │                      │                  │                      │
  └──────────┘    └──────────────────────┘                  └──────────────────────┘

- This diagram illustrates how raw data is transformed into Nautilus data structures.
```

Conceretely, this would involve:
- `BinanceOrderBookDeltaDataLoader.load(...)` which reads CSV files provided by Binance from disk, and returns a `pd.DataFrame`
- `OrderBookDeltaDataWrangler.process(...)` which takes the `pd.DataFrame` and returns `list[OrderBookDelta]`

The following example shows how to accomplish the above in Python:
```python
import os

from nautilus_trader import PACKAGE_ROOT
from nautilus_trader.persistence.loaders import BinanceOrderBookDeltaDataLoader
from nautilus_trader.persistence.wranglers import OrderBookDeltaDataWrangler
from nautilus_trader.test_kit.providers import TestInstrumentProvider


# Load raw data
data_path = os.path.join(PACKAGE_ROOT, "tests/test_data/binance-btcusdt-depth-snap.csv")
df = BinanceOrderBookDeltaDataLoader.load(data_path)

# Setup a wrangler
instrument = TestInstrumentProvider.btcusdt_binance()
wrangler = OrderBookDeltaDataWrangler(instrument)

# Process to a list `OrderBookDelta` Nautilus objects
deltas = wrangler.process(df)
```

## Data catalog

The data catalog is a central store for Nautilus data, persisted in the [Parquet](https://parquet.apache.org) file format.

We have chosen Parquet as the storage format for the following reasons:
- It performs much better than CSV/JSON/HDF5/etc in terms of compression ratio (storage size) and read performance
- It does not require any separate running components (for example a database)
- It is quick and simple to get up and running with

The Arrow schemas used for the Parquet format are either single sourced in the core `persistence` Rust library, or available
from the `/serialization/arrow/schema.py` module.

```{note}
2023-10-14: The current plan is to eventually phase out the Python schemas module, so that all schemas are single sourced in the Rust core.
```

### Initializing
The data catalog can be initialized from a `NAUTILUS_PATH` environment variable, or by explicitly passing in a path like object.

The following example shows how to initialize a data catalog where there is pre-existing data already written to disk at the given path.

```python
CATALOG_PATH = os.getcwd() + "/catalog"

# Create a new catalog instance
catalog = ParquetDataCatalog(CATALOG_PATH)
```

### Writing data
New data can be stored in the catalog, which is effectively writing the given data to disk in the Nautilus-specific Parquet format.
All Nautilus built-in `Data` objects are supported, and any data which inherits from `Data` can be written.

The following example shows the above list of Binance `OrderBookDelta` objects being written.
```python
catalog.write_data(deltas)
```

```{warning}
Existing data for the same data type, `instrument_id`, and date will be overwritten without prior warning.
Ensure you have appropriate backups or safeguards in place before performing this action.
```

Rust Arrow schema implementations and available for the follow data types (enhanced performance):
- `OrderBookDelta`
- `QuoteTick`
- `TradeTick`
- `Bar`

### Reading data
Any stored data can then we read back into memory:
```python
start = dt_to_unix_nanos(pd.Timestamp("2020-01-03", tz=pytz.utc))
end =  dt_to_unix_nanos(pd.Timestamp("2020-01-04", tz=pytz.utc))

deltas = catalog.order_book_deltas(instrument_ids=[instrument.id.value], start=start, end=end)
```

### Streaming data
When running backtests in streaming mode with a `BacktestNode`, the data catalog can be used to stream the data in batches.

The following example shows how to achieve this by initializing a `BacktestDataConfig` configuration object:
```python
data_config = BacktestDataConfig(
    catalog_path=str(catalog.path),
    data_cls=OrderBookDelta,
    instrument_id=instrument.id.value,
    start_time=start,
    end_time=end,
)
```

This configuration object can then be passed into a `BacktestRunConfig` and then in turn passed into a `BacktestNode` as part of a run.
See the [Backtest (high-level API)](../tutorials/backtest_high_level.md) tutorial for more details.
