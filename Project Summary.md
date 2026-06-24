Project Summary
The objective of this project is to develop an algorithmic trading engine designed to operate in the 0DTE (Zero Days to Expiration) options market. The system follows a structured, multi-stage workflow to identify and execute high-probability trades.
Core Processes: 

Opening Range Identification: Define the initial price range during the opening period.
Breakout Detection: Monitor and detect valid breakouts from the established opening range.
Gamma Exposure (GEX) Analysis: Calculate and incorporate GEX metrics to enhance trade selection.
Trade Execution: Systematically open and close positions based on signal confirmation.


**Stage 1: Opening Range Identification**

The goal is to define the initial price range during the first 30 minutes of the market session.

Step 1: IB API Communication Flow (High-Level)

Step,API Client Action (You Send),TWS/IB Gateway Action (You Receive)
Setup,"1. Connect to TWS/IB Gateway (e.g., eClient.connect(...)).","2. Connection established (e.g., eWrapper.connectionTime(...) is called)."
Contract,"3. Define the contract for your ticker (e.g., Contract object: symbol, secType, exchange, currency).","No immediate response, this is local."
Request Data,"4. Send the historical data request (e.g., eClient.reqHistoricalData(...)).","5. Data received in bars (e.g., eWrapper.historicalData(...) is called repeatedly for each bar)."
Data Received,,"6. End of data stream (e.g., eWrapper.historicalDataEnd(...) is called to signal completion)."
Cleanup,"7. Disconnect (e.g., eClient.disconnect()).",Connection is closed.

Step 2: Request Parameters (First 30 Minutes)

contract: The ticker of interest (e.g., AAPL).
endDateTime: Set to the current date and 30 minutes after market open (e.g., 09:30:00 EST + 30 mins).
durationStr: Set to '30 min' to define the historical data window.
barSizeSetting: Set to '1 min' resolution to accurately capture high/low.
whatToShow: Use 'TRADES' for open, high, low, close, and volume.
useRTH: Set to 1 (True) to ignore pre-market data

Step 3: Calculating Levels

Store Bar Data: Collect High and Low prices for every 1-minute bar received.
Filter/Validate: Ensure bars strictly fall within the market open window (no pre-market).
Find Range: Once historicalDataEnd is received:
HIGH_LEVEL: The maximum value of all collected High prices.
LOW_LEVEL: The minimum value of all collected Low prices.


**Stage 2: Breakout Detection**

Monitor real-time data to detect clean breakouts from the established range.

Step 1: Real-Time Data Flow
Setup: Ensure the TWS/Gateway connection is active.
Request Data: Send a request using eClient.reqRealTimeBars(...). Receive data via eWrapper.realtimeBar(...) every time a bar closes.
Processing: Client-side code aggregates 5-second bars into 5-minute candles and applies logic.
Cleanup: Cancel the subscription using eClient.cancelRealTimeBars(...) when finished.

Step 2: Breakout Calculation Conditions

Let the 5-minute candle consist of: $C_{High}$, $C_{Low}$, $C_{Open}$, and $C_{Close}$.

High Breakout (Bullish Signal):
Direction: Must be a bullish candle ($C_{Close} > C_{Open}$)
Clean Break: The entire body must be above the level ($C_{Low} > HIGH\_LEVEL$)

Low Breakout (Bearish Signal):
Direction: Must be a bearish candle ($C_{Close} < C_{Open}$)
Clean Break: The entire body must be below the level ($C_{High} < LOW\_LEVEL$)

Signal Result:

If High Breakout is true: BULLISH.
If Low Breakout is true: BEARISH.
Otherwise: NO SIGNAL.


**Stage 3: Gamma Exposure (GEX) Analysis**

Objective: create a third stage where we will get the gamma exposure in three different ways, to be used in the four stage: 

    * We will start creating a parameter in the config.yaml where we are going to be able to select the type of process that calculate the GEX
        The parameter will have: 

            1. Name: GEX Process type
            2. it needs to be initialized as 0, but can be 0,1 or 2 

    * If GEX Process type = 0 the program will use the Option 1 Use a Third-Party GEX Service (gexbot)
    * If GEX Process type = 1 the program will use the Option 2 Build a Parallelized "GEX Microservice"
    * If GEX Process type = 2 the program will use the Option 3 Use a Dedicated Data Feed

    # Option 1 Use a Third-Party GEX Service (gexbot) - Process Specification: GEX Calculation via Third-Party Service (gexbot)

        This document outlines the technical specification for **Option 1: Use a Third-Party GEX Service (gexbot)**.
        The process leverages the `gexbot.com` REST API to acquire gamma distribution data.

    **Objective:** To identify the single options strike price with the highest absolute concentration of gamma exposure for a given underlying and expiration. 
        This strike level, referred to as `MAX_GAMMA_STRIKE`, will be used as a key input for the trade validation logic in Stage 4.

        ---

        ### Step 1: Connection & Authentication

        **Objective:** Establish a secure, authenticated session with the `gexbot.com` API.

        1.  **Action:** The system will initiate a secure HTTPS connection to the API's base endpoint.
        2.  **Authentication:** Every request must include the API Key as a query parameter in the URL.
            *   **URL Format:** `https://api.gexbot.com/v1/...?api_key=<YOUR_API_KEY>`
        3.  **Parameters Required (from `config.yaml`):**
            *   `gexbot_api_key`: The secret API key for the `gexbot.com` service.
            *   `gexbot_base_url`: The root URL for the API (`https://api.gexbot.com/`).
        4.  **Error Handling:**
            *   If a `401 Unauthorized` response is received, the connection has failed due to an invalid API key. The system should log a critical error and halt the GEX calculation process.
            *   Connection timeouts or other network-level errors should trigger a retry mechanism before failing.

        ---

        ### Step 2: Data Retrieval (Gamma Fetch)

        **Objective:** Fetch the raw gamma distribution data for a specific ticker and expiration date in a single API call.

        1.  **Action:** The system will send a single `GET` request to the gamma distribution endpoint.
        2.  **Endpoint:** `/v1/gex/distribution`
        3.  **Request Parameters:**
            *   `api_key` (string, required): The authentication key.
            *   `ticker` (string, required): The underlying symbol (e.g., "SPX").
            *   `exp` (string, required): The target expiration date in `YYYY-MM-DD` format. The system will construct this date based on the `days_to_expiration` setting in `config.yaml`.
        4.  **Expected Response (JSON):** The API will return a JSON object containing a `data` array. Each element in the array represents a single strike and its associated gamma values.

            ```json
            {
            "success": true,
            "data": [
                {
                "strike": 5000,
                "long_gamma": 1234567.89,
                "short_gamma": -987654.32,
                "total_gamma": 246913.57,
                "vanna": ...,
                "charm": ...
                },
                {
                "strike": 5010,
                "long_gamma": 1100000.00,
                "short_gamma": -1050000.00,
                "total_gamma": 50000.00,
                ...
                },
                // ... more strikes
            ]
            }
            ```

        ---

    ### Step 3: Aggregation (Total GEX Calculation)

        **Objective:** To determine the total magnitude of gamma risk at each strike, irrespective of whether it comes from long or short positions.

        1.  **Logic:** For every unique strike `K` returned by the API, the system calculates the **Total Gamma Exposure** by summing the absolute values of the long and short gamma.

            `Total_GEX_K = ABS(Short_Gamma_K) + ABS(Long_Gamma_K)`

        2.  **Process:** The system will iterate through the `data` array from the API response. 
        For each strike, it will perform the absolute summation and store the result in a dictionary where the key is the strike price and the value is its calculated `Total_GEX`.

        *(Note: The `total_gamma` field from the API response is ignored, as the bot's logic requires summing the magnitudes, not using the net value).*


        ---

        ### Step 4: Max GEX Identification

        **Objective:** Isolate the single strike price with the highest concentration of total gamma to be used in the trading logic.

        1.  **Operation:** The system will perform a "max" search on the aggregated GEX results from Step 3. It will identify the key (strike) that corresponds to the largest value (Total GEX).
        2.  **Variable Assignment:** The strike price with the highest `Total_GEX` value is stored in a variable for use in Stage 4.

        **Final Output:** The provider returns a tuple containing the `max_gamma_strike` (float), the `expiration_date` (string in 'YYYYMMDD' format), and a list of the `strikes` that were analyzed. This complete dataset is then consumed by the `Trade Execution` stage.


    Option 2: Build a Parallelized "GEX Microservice" - GEX Process type = 1 

     # Specification: Interactive Brokers GEX Provider (`IBProvider`)

        ## 1. Introduction & Purpose

        This document outlines the specification for creating a new Gamma Exposure (GEX) data provider, the `IBProvider`. This provider will calculate GEX by fetching real-time options and market data directly from Interactive Brokers (IB).

        The primary goals of this provider are:
        - To offer a self-sufficient, cost-effective method for calculating GEX without relying on third-party paid APIs.
        - To increase the application's robustness by providing an alternative GEX source.
        - To integrate seamlessly into the existing provider factory, adhering to the project's architecture.

        ## 2. File and Class Structure

        - **New File:** `strategy/gex/ib_provider.py`
        - **Class Name:** `IBProvider`
        - **Inheritance:** The `IBProvider` class must inherit from `strategy.gex.base_provider.BaseGexProvider` and implement all its abstract methods.

        ## 3. `IBProvider` Implementation Details

        ### 3.1. Initialization (`__init__`)

        The constructor will accept the application's `config` object.

        ```python
        # Conceptual signature
        def __init__(self, config: Config):
            # ...
        ```

        - It will be responsible for retrieving the singleton instance of the `IBConnector` from the `ib_client.connector` module to enable interaction with the TWS/Gateway API.
        - The `get_max_gamma_strike` method will accept the `ib_connector` instance as an argument.

        ### 3.2. Main Method (`get_max_gamma_strike`)

        This method is the core of the provider and must adhere to the `BaseGexProvider` interface.

        ```python
        # Conceptual signature
        def get_max_gamma_strike(self, ticker: str, ib_connector: "IBConnector") -> tuple[float, str, list[float]]:
            # ...
        ```

        **Functional Requirements:**

        1.  **Fetch Option Chain:**
            - Use the `IBConnector` to fetch the option chain for the given underlying `symbol`.
            - The fetch request must be filtered based on parameters from `config.yaml`:
                - `gex.days_to_expiration`: This will control how far out the expiration dates for the options contracts are.
                - `gex.strikes_quantity`: This will control how many strikes around the money are fetched.
            - This includes fetching all available expiration dates and strikes for both calls and puts within the defined filters.
        2.  **Batch Market Data Request:**
            - To ensure efficiency, the provider must request model greeks (specifically Gamma) for all relevant option contracts in a single, batched/concurrent request. It must not loop and request data for each contract sequentially.
        4.  **Calculate Gamma Exposure (GEX):**
            - For each strike price, calculate the total GEX.
            - The GEX for a single option contract is: `GEX = gamma * open_interest * 100 * option_volume`.
            - The total GEX for a strike is the sum of the GEX of all contracts at that strike.
            - **Calculation Logic:** As per the provided analysis, the GEX contribution from both call and put options should be aggregated to find the total magnitude. The specified logic is `call_gex + put_gex`. The implementation must sum the calculated GEX values for calls and puts at each strike.
        5.  **Identify Max Gamma Strike:**
            - After calculating the total GEX for every strike price in the chain, identify the strike with the maximum GEX value.
        6.  **Return Value:**
            - Return a tuple containing the `max_gamma_strike` (float), the `expiration_date` (string), and the list of `strikes` analyzed.
            - Return `(0.0, "", [])` if the data cannot be fetched or if no options exist for the symbol.

        ## 4. Integration with Provider Factory and Configuration

        The `strategy/gex/factory.py` file must be updated to integrate the new provider.

        - **Import:** Add `from .ib_provider import IBProvider` to the top of the file.
        - **Update Factory Function:** Modify the `get_gex_provider` function to instantiate `IBProvider` when `config.gex.provider_type` is `1`, replacing the `NotImplementedError`.

        ```python
        # strategy/gex/factory.py - Target state change
        ...
        from .ib_provider import IBProvider # Import the new provider

        def get_gex_provider(config: Config) -> BaseGexProvider:
            provider_type = config.gex.provider_type

            if provider_type == 0:
                return GexbotProvider(config)
            elif provider_type == 1:
                # Now implemented with the refactored code
                return IBProvider(config) 
            elif provider_type == 2:
                return MassiveDataProvider(config)
        # ... other gex settings
        ```

        ## 6. Testing Requirements

        A new test suite must be created to ensure the `IBProvider` functions correctly.

        - **New Test File:** `tests/strategy/gex/test_ib_provider.py`
        - **Test Cases:**
            1.  **Factory Test:** Verify that `get_gex_provider` returns an `IBProvider` instance when `provider_type` is set to `1`.
            2.  **GEX Calculation Test:**
                - Create a unit test that calls the GEX calculation logic directly.
                - Use a predefined set of mock option contracts with known `gamma` and `open_interest`.
                - Assert that the calculated GEX for each strike and the final max gamma strike are correct, specifically verifying the `call_gex + put_gex` summation logic.
            3.  **End-to-End Method Test (`get_max_gamma_strike`):**
                - Mock the `IBConnector` instance and its data-fetching methods.
                - Simulate the IB API returning a sample option chain and market data.
                - Assert that `get_max_gamma_strike` returns the expected strike price.
                - Verify that the implementation attempts to fetch market data concurrently.
            4.  **Error Handling Tests:**
                - Simulate the IB API returning no options for a symbol. Assert the method returns `None`.
                - Simulate API errors during data fetching. Assert that errors are handled gracefully and the method returns `None`.

    Option 3: Use a Dedicated Data Feed - GEX Process type = 2

        # Process Specification: Feed Providers for GEX Calculation (massive)

    This document outlines the technical specification for **Option 3: Use a Dedicated Data Feed** to calculate the strike with the maximum Gamma Exposure (GEX). 
    This process leverages the `massive.com` REST API to acquire the necessary options data in a single, efficient snapshot.

    **Objective:** To identify the single options strike price with the highest absolute concentration of gamma exposure for a given underlying and expiration. 
    This strike level, referred to as `MAX_GAMMA_STRIKE`, serves as a key input for the trade execution logic in Stage 4.

    ---

    ### Step 1: Connection & Authentication

    **Objective:** Establish a secure, authenticated session with the `massive.com` API.

    1.  **Action:** The system will initiate a secure HTTPS connection to the API's base endpoint.
    2.  **Authentication:** Every request must include an `Authorization` header containing the API Key provided by `massive.com`.
        *   **Header Format:** `Authorization: Bearer <YOUR_API_KEY>`
    3.  **Parameters Required (from `config.yaml`):**
        *   `api_key`: The secret API key for the `massive.com` service.
        *   `base_url`: The root URL for the API (e.g., `https://api.massive.com/v1`).
    4.  **Error Handling:**
        *   If a `401 Unauthorized` response is received, the connection attempt has failed due to an invalid API key. The system should log a critical error and halt the GEX calculation process.
        *   Connection timeouts or other network-level errors should trigger a retry mechanism (e.g., retry up to 3 times) before failing.

    ---

    ### Step 2: Data Request (Parameter-Based Snapshot)

    **Objective:** Fetch a complete snapshot of all required options data (quotes, greeks, open interest) in a single API call to minimize latency.

    1.  **Action:** The system will send a single `GET` request to the options chain endpoint.
    2.  **Endpoint:** `/options/chain` (example endpoint based on documentation).
    3.  **Request Parameters:**
        *   `ticker` (string): The underlying symbol (e.g., "SPX").
        *   `days_to_expiration` (integer): The target number of days until the option's expiration.
            The API will use this to select the single, closest expiration date (e.g., a value of `0` targets the 0DTE chain).
        *   `strikes_quantity` (integer): The total number of strikes to retrieve, centered around the current at-the-money (ATM) price. 
            For example, a value of `120` would return the 60 strikes above and 60 strikes below the ATM price.
        *   `fields` (string): A comma-separated list specifying the exact data points to return. This ensures the payload is lean.
            *   **Required Value:** `"greeks,openInterest"`
    4.  **Expected Response (JSON):** The API should return a JSON object containing a list of option contracts for the requested chain. 
        Each element in the list represents a single option (a call or a put) and its associated data.

        ```json
        {
        "expiration": "2026-01-21",
        "options": [
            {
            "strike": 5000.0,
            "type": "call",
            "openInterest": 1520,
            "greeks": { "gamma": 0.0015, "delta": 0.52, ... }
            },
            {
            "strike": 5000.0,
            "type": "put",
            "openInterest": 2100,
            "greeks": { "gamma": 0.0014, "delta": -0.48, ... }
            },
            // ... more contracts
        ]
        }
        ```

    ---

    ### Step 3: GEX Calculation and Aggregation

    **Objective:** Process the raw data from the API to calculate the total gamma exposure for each individual strike price.

    1.  **Gamma Exposure Calculation:** The system iterates through every option contract (both calls and puts) in the API response. For each contract, it calculates its individual gamma exposure.
        *   `Gamma_Exposure = option.greeks.gamma * option.open_interest * option_multiplier`

        *(Note: The `option_multiplier` is typically 100 and is sourced from `config.yaml`)*

    2.  **Aggregation by Strike:** The system aggregates these individual GEX values into a dictionary where the key is the strike price. It sums the `Gamma_Exposure` from both the call and the put at that strike to get the total.

        `Total_GEX_at_Strike = Call_GEX + Put_GEX`

        **Example Result:**
        ```
        {
        5000: 22500,
        5010: 18700,
        5020: 31400,  // High concentration
        ...
        }
        ```

    ---

    ### Step 5: Max GEX Identification

    **Objective:** Isolate the single strike price with the highest concentration of total gamma to be used in the trading logic.

    1.  **Operation:** The system will perform a "max" search on the aggregated GEX results. It will find the key (strike) corresponding to the largest value (Total GEX).
    2.  **Variable Assignment:** The strike price with the highest `Total_GEX` value is stored in a variable for use in Stage 4.

    **Final Output:** The provider returns a tuple containing the `max_gamma_strike` (float) and the `expiration_date` (string in 'YYYYMMDD' format). This dataset is then consumed by the `Trade Execution` stage.


**Stage 4: Trade Execution**

# Trade Execution Specification

This document outlines the step-by-step process for trade execution within the bot.

## Process Overview

When the `Engine` calls `OrderManager.place_trade()`, the following sequence of events is triggered:

### 1. Trade Validation (`_make_trade_decision`)

This is the final checkpoint before a trade is initiated. It validates the trade signal against market conditions.

*   **Inputs from Breakout Process:**
    *   `signal_type`: The direction of the signal (e.g., 'Buy', 'Sell').
    *   `spot_price`: The current market price of the underlying asset.
*   **Inputs from GEX Process:**
    *   A tuple containing:
        *   `strike_price`: The relevant strike price from GEX analysis.
        *   `date`: The expiration date for the option.

*   **Execution Conditions:** The decision to execute a specific option type is based on the following logic:

| Signal Type | Condition                 | Action           |
|-------------|---------------------------|------------------|
| Buy         | `strike_price > spot_price` | Long Call @ ATM  |
| Buy         | `strike_price < spot_price` | Long Put @ ATM   |
| Sell        | `strike_price < spot_price` | Long Put @ ATM   |
| Sell        | `strike_price > spot_price` | Long Call @ ATM  |

### 2. ATM Strike Calculation (`_get_atm_strike`)

If the trade is validated, this step calculates the At-The-Money (ATM) strike price.

*   **Logic:** The `spot_price` is rounded to the nearest available strike price. The rounding direction might be specified as "above", but for ATM it should be the closest. *Initial implementation will round to the nearest strike above.*

### 3. Option Contract Definition (`_create_option_contract`)

This step constructs the official IBKR `Contract` object required for placing an order.

*   **Inputs:**
    *   ATM Strike Price (from step 2).
    *   Trade Direction (Call/Put, determined in step 1).
    *   Expiration Date (from GEX process).
*   **Output:** An `ibapi.contract.Contract` object fully defined for the trade.

### 4. Fetch Option Price

Before submitting the order, the system must fetch the current market price of the specific option contract defined in the previous step.

*   **Action:** Request market data for the created `Contract` object form ibkr.
*   **Output:** The current bid/ask or last traded price of the option.

### 5. Place Opening Order

This two-part step creates and submits the opening order to IBKR.

1.  **Create Opening Order:** An `ibapi.order.Order` object is created. This will specify order type (e.g., Market, Limit), quantity, and action (BUY).
2.  **Submit Opening Order:** The `Contract` and `Order` objects are submitted to IBKR via the API.

### 6. Post-Trade Management

This stage begins immediately after the trade is submitted and is managed by the `Engine` in the `MANAGING_TRADE` state.

*   **Atomic Bracket Order Submission:**
    1.  **Calculate TP/SL Prices:** Based on the parent order's limit price, the system calculates the absolute price levels for the Take Profit (TP) and Stop Loss (SL) using percentage values from `config.yaml`. Prices are carefully rounded to the correct tick size for the instrument.
    2.  **Structure Bracket Orders:** The `OrderManager` creates three `Order` objects:
        *   **Parent Order:** The main entry order (e.g., LMT), with `Transmit=False`.
        *   **Take Profit Order:** A `LMT` order with its `parentId` set to the parent's ID, also with `Transmit=False`.
        *   **Stop Loss Order:** A `STP` order with its `parentId` set to the parent's ID, but with `Transmit=True`.
    3.  **Atomic Submission:** The three orders are sent to IBKR sequentially. The final order with `Transmit=True` causes the entire group to be submitted atomically, ensuring the position is protected by a server-side bracket from the moment of execution.

*   **Milestone-based Trailing Stop Loss:**
    *   **Monitoring:** Once the parent order is filled, the `Engine` enters the `MANAGING_TRADE` state, periodically calling `OrderManager.manage_open_positions()`.
    *   **Trailing Condition:** The trailing stop logic is activated when the position's profit crosses a new "milestone" defined by the `activation_profit_pct` from `config.yaml`. For example, if `activation_profit_pct` is 10, milestones are crossed at 10%, 20%, 30%, etc.
    *   **Trailing Action (Order Modification):**
        *   When a new milestone is crossed, the system calculates a new, improved stop-loss price.
        *   The logic reduces the initial risk (`stop_loss_pct`) by a factor of `trail_pct` for each milestone achieved.
        *   **Example:** If initial risk is 50% and `trail_pct` is 16%, after the first milestone, the new risk becomes `50% - 16% = 34%`. The new stop price is calculated based on this reduced risk from the entry price.
        *   The system then **modifies** the existing Stop Loss order by submitting a new order with the **same `orderId`** but the new, higher stop price. This updates the order on the broker's server without affecting the Take Profit order.

*   **Position Closure:**
    *   The `OrderManager` monitors the order status queue. When it detects that either the Take Profit or Stop Loss order has been filled or cancelled, it considers the position closed.
    *   The position is removed from active management, and the `Engine` transitions to the `SHUTDOWN` state.

## Configuration Parameters (`config.yaml`)

The following parameters will be required in `config.yaml` to support this process:

```yaml
trade_execution:
  total_quantity: 1
  order_defaults:
    entry_order_type: "LMT"
    tp_order_type: "LMT"
    sl_order_type: "STP"

trade_management:
  take_profit_pct: 40
  stop_loss_pct: 50
  trailing_stop:
    activation_profit_pct: 10
    trail_pct: 16