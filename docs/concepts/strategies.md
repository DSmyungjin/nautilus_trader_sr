# Strategies

The heart of the NautilusTrader user experience is in writing and working with
trading strategies. Defining a trading strategy is achieved by inheriting the `Strategy` class, 
and implementing the methods required by the users trading strategy logic.

Using the basic building blocks of data ingest, event handling, and order management (which we will discuss
below), it's possible to implement any type of trading strategy including directional, momentum, re-balancing,
pairs, market making etc.

Refer to the `Strategy` in the [API Reference](../api_reference/trading.md) for a complete description
of all available methods.

There are two main parts of a Nautilus trading strategy:
- The strategy implementation itself, defined by inheriting the `Strategy` class
- The _optional_ strategy configuration, defined by inheriting the `StrategyConfig` class

```{note}
Once a strategy is defined, the same source can be used for backtesting and live trading.
```

The main capabilities of a strategy include:
- Historical data requests
- Live data feed subscriptions
- Setting time alerts or timers
- Cache access
- Portfolio access
- Creating and managing orders and positions

## Implementation
Since a trading strategy is a class which inherits from `Strategy`, you must define
a constructor where you can handle initialization. Minimally the base/super class needs to be initialized:

```python
class MyStrategy(Strategy):
    def __init__(self) -> None:
        super().__init__()  # <-- the super class must be called to initialize the strategy
```

From here, you can implement handlers as necessary to perform actions based on state transitions
and events.

### Handlers

Handlers are methods within the `Strategy` class which may perform actions based on different types of events or state changes.
These methods are named with the prefix `on_*`. You can choose to implement any or all of these handler 
methods depending on the specific needs of your strategy.

The purpose of having multiple handlers for similar types of events is to provide flexibility in handling granularity. 
This means that you can choose to respond to specific events with a dedicated handler, or use a more generic
handler to react to a range of related events (using switch type logic). The call sequence is generally most specific to most general.

#### Stateful actions

These handlers are triggered by lifecycle state changes of the `Strategy`. It's recommended to:

- Use the `on_start` method to initialize your strategy (e.g., fetch instruments, subscribe to data)
- Use the `on_stop` method for cleanup tasks (e.g., cancel open orders, close open positions, unsubscribe from data)

```python
def on_start(self) -> None:
def on_stop(self) -> None:
def on_resume(self) -> None:
def on_reset(self) -> None:
def on_dispose(self) -> None:
def on_degrade(self) -> None:
def on_fault(self) -> None:
def on_save(self) -> dict[str, bytes]:  # Returns user defined dictionary of state to be saved
def on_load(self, state: dict[str, bytes]) -> None:
```

#### Data handling

These handlers deal with market data updates.
You can use these handlers to define actions upon receiving new market data.

```python
def on_order_book_deltas(self, deltas: OrderBookDeltas) -> None:
def on_order_book(self, order_book: OrderBook) -> None:
def on_ticker(self, ticker: Ticker) -> None:
def on_quote_tick(self, tick: QuoteTick) -> None:
def on_trade_tick(self, tick: TradeTick) -> None:
def on_bar(self, bar: Bar) -> None:
def on_venue_status(self, data: VenueStatus) -> None:
def on_instrument(self, instrument: Instrument) -> None:
def on_instrument_status(self, data: InstrumentStatus) -> None:
def on_instrument_close(self, data: InstrumentClose) -> None:
def on_historical_data(self, data: Data) -> None:
def on_data(self, data: Data) -> None:  # Generic data passed to this handler
```

#### Order management

Handlers in this category are triggered by events related to orders.
`OrderEvent` type messages are passed to handlers in the following sequence:

1. Specific handler (e.g., `on_order_accepted`, `on_order_rejected`, etc.)
2. `on_order_event(...)`
3. `on_event(...)`

```python
def on_order_initialized(self, event: OrderInitialized) -> None:
def on_order_denied(self, event: OrderDenied) -> None:
def on_order_emulated(self, event: OrderEmulated) -> None:
def on_order_released(self, event: OrderReleased) -> None:
def on_order_submitted(self, event: OrderSubmitted) -> None:
def on_order_rejected(self, event: OrderRejected) -> None:
def on_order_accepted(self, event: OrderAccepted) -> None:
def on_order_canceled(self, event: OrderCanceled) -> None:
def on_order_expired(self, event: OrderExpired) -> None:
def on_order_triggered(self, event: OrderTriggered) -> None:
def on_order_pending_update(self, event: OrderPendingUpdate) -> None:
def on_order_pending_cancel(self, event: OrderPendingCancel) -> None:
def on_order_modify_rejected(self, event: OrderModifyRejected) -> None:
def on_order_cancel_rejected(self, event: OrderCancelRejected) -> None:
def on_order_updated(self, event: OrderUpdated) -> None:
def on_order_filled(self, event: OrderFilled) -> None:
def on_order_event(self, event: OrderEvent) -> None:  # All order event messages are eventually passed to this handler
```

#### Position management

Handlers in this category are triggered by events related to positions.
`PositionEvent` type messages are passed to handlers in the following sequence:

1. Specific handler (e.g., `on_position_opened`, `on_position_changed`, etc.)
2. `on_position_event(...)`
3. `on_event(...)`

```python
def on_position_opened(self, event: PositionOpened) -> None:
def on_position_changed(self, event: PositionChanged) -> None:
def on_position_closed(self, event: PositionClosed) -> None:
def on_position_event(self, event: PositionEvent) -> None:  # All position event messages are eventually passed to this handler
```

#### Generic event handling

This handler will eventually receive all event messages which arrive at the strategy, including those for
which no other specific handler exists.

```python
def on_event(self, event: Event) -> None:
```

#### Handler example

The following example shows a typical `on_start` handler method implementation (taken from the example EMA cross strategy).
Here we can see the following:
- Indicators being registered to receive bar updates
- Historical data being requested (to hydrate the indicators)
- Live data being subscribed to

```python
def on_start(self) -> None:
    """
    Actions to be performed on strategy start.
    """
    self.instrument = self.cache.instrument(self.instrument_id)
    if self.instrument is None:
        self.log.error(f"Could not find instrument for {self.instrument_id}")
        self.stop()
        return

    # Register the indicators for updating
    self.register_indicator_for_bars(self.bar_type, self.fast_ema)
    self.register_indicator_for_bars(self.bar_type, self.slow_ema)

    # Get historical data
    self.request_bars(self.bar_type)

    # Subscribe to live data
    self.subscribe_bars(self.bar_type)
    self.subscribe_quote_ticks(self.instrument_id)
```

### Clock and timers

Strategies have access to a comprehensive `Clock` which provides a number of methods for creating
different timestamps, as well as setting time alerts or timers.

```{note}
See the `Clock` [API reference](../api_reference/common.md#Clock) for a complete list of available methods.
```

#### Current timestamps

While there are multiple ways to obtain current timestamps, here are two commonly used methods as examples:

**UTC Timestamp:** This method returns a timezone-aware (UTC) timestamp:
```python
now: pd.Timestamp = self.clock.utc_now()
```

**Unix Nanoseconds:** This method provides the current timestamp in nanoseconds since the UNIX epoch:
```python
unix_nanos: int = self.clock.timestamp_ns()
```

#### Time alerts

Time alerts can be set which will result in a `TimeEvent` being dispatched to the `on_event` handler at the
specified alert time. In a live context, this might be slightly delayed by a few microseconds.

This example sets a time alert to trigger one minute from the current time:
```python
self.clock.set_alert_time(
    name="MyTimeAlert1",
    alert_time=self.clock.utc_now() + pd.Timedelta(minutes=1),
)
```

#### Timers

Continuous timers can be setup which will generate a `TimeEvent` at regular intervals until the timer expires
or is canceled.

This example sets a timer to fire once per minute, starting immediately:
```python
self.clock.set_timer(
    name="MyTimer1",
    interval=pd.Timedelta(minutes=1),
)
```

### Cache access

The traders central `Cache` can be accessed to fetch data and execution objects (orders, positions etc).
There are many methods available often with filtering functionality, here we go through some basic use cases.

#### Fetching data

The following example shows how data can be fetched from the cache (assuming some instrument ID attribute is assigned):

```python
last_quote = self.cache.quote_tick(self.instrument_id)
last_trade = self.cache.trade_tick(self.instrument_id)
last_bar = self.cache.bar(<SOME_BAR_TYPE>)
```

#### Fetching execution objects

The following example shows how individual order and position objects can be fetched from the cache:

```python
order = self.cache.order(<SOME_CLIENT_ORDER_ID>)
position = self.cache.position(<SOME_POSITION_ID>)

```

Refer to the `Cache` in the [API Reference](../api_reference/cache.md) for a complete description
of all available methods.

### Portfolio access

The traders central `Portfolio` can be accessed to fetch account and positional information.
The following shows a general outline of available methods.

#### Account and positional information

```python
def account(self, venue: Venue) -> Account

def balances_locked(self, venue: Venue) -> dict[Currency, Money]
def margins_init(self, venue: Venue) -> dict[Currency, Money]
def margins_maint(self, venue: Venue) -> dict[Currency, Money]
def unrealized_pnls(self, venue: Venue) -> dict[Currency, Money]
def net_exposures(self, venue: Venue) -> dict[Currency, Money]

def unrealized_pnl(self, instrument_id: InstrumentId) -> Money
def net_exposure(self, instrument_id: InstrumentId) -> Money
def net_position(self, instrument_id: InstrumentId) -> decimal.Decimal

def is_net_long(self, instrument_id: InstrumentId) -> bool
def is_net_short(self, instrument_id: InstrumentId) -> bool
def is_flat(self, instrument_id: InstrumentId) -> bool
def is_completely_flat(self) -> bool
```

Refer to the `Portfolio` in the [API Reference](../api_reference/portfolio.md) for a complete description
of all available methods.

#### Reports and analysis

The `Portfolio` also makes a `PortfolioAnalyzer` available, which can be fed with a flexible amount of data 
(to accommodate different lookback windows). The analyzer can provide tracking for and generating of performance
metrics and statistics.

Refer to the `PortfolioAnalyzer` in the [API Reference](../api_reference/analysis.md) for a complete description
of all available methods.

```{tip}
Also see the [Porfolio statistics](../concepts/advanced/portfolio_statistics.md) guide.
```

### Trading commands

NautilusTrader offers a comprehensive suite of trading commands, enabling granular order management 
tailored for algorithmic trading. These commands are essential for executing strategies, managing risk, 
and ensuring seamless interaction with various trading venues. In the following sections, we will 
delve into the specifics of each command and its use cases.

#### Submitting orders

An `OrderFactory` is provided on the base class for every `Strategy` as a convenience, reducing
the amount of boilerplate required to create different `Order` objects (although these objects
can still be initialized directly with the `Order.__init__(...)` constructor if the trader prefers).

The component an order flows to when submitted for execution depends on the following:

- If an `emulation_trigger` is specified, the order will _firstly_ be sent to the `OrderEmulator`
- If an `exec_algorithm_id` is specified (with no `emulation_trigger`), the order will _firstly_ be sent to the relevant `ExecAlgorithm` (assuming it exists and has been registered correctly)
- Otherwise, the order will _firstly_ be sent to the `RiskEngine`

The following examples show method implementations for a `Strategy`.

This example submits a `LIMIT` BUY order for emulation (see [OrderEmulator](advanced/emulated_orders.md)):
```python
    def buy(self) -> None:
        """
        Users simple buy method (example).
        """
        order: LimitOrder = self.order_factory.limit(
            instrument_id=self.instrument_id,
            order_side=OrderSide.BUY,
            quantity=self.instrument.make_qty(self.trade_size),
            price=self.instrument.make_price(5000.00),
            emulation_trigger=TriggerType.LAST_TRADE,
        )

        self.submit_order(order)
```

```{note}
It's possible to specify both order emulation, and an execution algorithm.
```

This example submits a `MARKET` BUY order to a TWAP execution algorithm:
```python
    def buy(self) -> None:
        """
        Users simple buy method (example).
        """
        order: MarketOrder = self.order_factory.market(
            instrument_id=self.instrument_id,
            order_side=OrderSide.BUY,
            quantity=self.instrument.make_qty(self.trade_size),
            time_in_force=TimeInForce.FOK,
            exec_algorithm_id=ExecAlgorithmId("TWAP"),
            exec_algorithm_params={"horizon_secs": 20, "interval_secs": 2.5},
        )

        self.submit_order(order)
```

#### Managed GTD expiry

It's possible for the strategy to manage expiry for orders with a time in force of GTD (_Good 'till Date_).
This may be desirable if the exchange/broker does not support this time in force option, or for any
reason you prefer the strategy to manage this.

To use this option, pass `manage_gtd_expiry=True` to your `StrategyConfig`. When an order is submitted with
a time in force of GTD, the strategy will automatically start an internal time alert.
Once the internal GTD time alert is reached, the order will be canceled (if not already closed).

Some venues (such as Binance Futures) support the GTD time in force, so to avoid conflicts when using
`managed_gtd_expiry` you should set `use_gtd=False` for your execution client config.

## Configuration

The main purpose of a separate configuration class is to provide total flexibility
over where and how a trading strategy can be instantiated. This includes being able
to serialize strategies and their configurations over the wire, making distributed backtesting
and firing up remote live trading possible.

This configuration flexibility is actually opt-in, in that you can actually choose not to have
any strategy configuration beyond the parameters you choose to pass into your
strategies' constructor. However, if you would like to run distributed backtests or launch
live trading servers remotely, then you will need to define a configuration.

Here is an example configuration:

```python
from decimal import Decimal
from nautilus_trader.config import StrategyConfig
from nautilus_trader.model.identifiers import InstrumentId
from nautilus_trader.trading.strategy import Strategy


class MyStrategyConfig(StrategyConfig):
    instrument_id: str
    bar_type: str
    fast_ema_period: int = 10
    slow_ema_period: int = 20
    trade_size: Decimal
    order_id_tag: str

# Here we simply add an instrument ID as a string, to 
# parameterize the instrument the strategy will trade.

class MyStrategy(Strategy):
    def __init__(self, config: MyStrategyConfig) -> None:
        super().__init__(config)

        # Configuration
        self.instrument_id = InstrumentId.from_str(config.instrument_id)


# Once a configuration is defined and instantiated, we can pass this to our 
# trading strategy to initialize.

config = MyStrategyConfig(
    instrument_id="ETHUSDT-PERP.BINANCE",
    bar_type="ETHUSDT-PERP.BINANCE-1000-TICK[LAST]-INTERNAL",
    trade_size=Decimal(1),
    order_id_tag="001",
)

strategy = MyStrategy(config=config)

```

```{note}
Even though it often makes sense to define a strategy which will trade a single
instrument. The number of instruments a single strategy can work with is only limited by machine resources.
```

### Multiple strategies

If you intend running multiple instances of the same strategy, with different
configurations (such as trading different instruments), then you will need to define
a unique `order_id_tag` for each of these strategies (as shown above).

```{note}
The platform has built-in safety measures in the event that two strategies share a
duplicated strategy ID, then an exception will be raised that the strategy ID has already been registered.
```

The reason for this is that the system must be able to identify which strategy
various commands and events belong to. A strategy ID is made up of the
strategy class name, and the strategies `order_id_tag` separated by a hyphen. For
example the above config would result in a strategy ID of `MyStrategy-001`.

```{tip}
See the `StrategyId` [documentation](../api_reference/model/identifiers.md) for further details.
```

