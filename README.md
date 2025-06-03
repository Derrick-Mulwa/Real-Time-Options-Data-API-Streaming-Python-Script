# Real-Time Options Data Streaming from Schwab API Python Script

This script is a robust Python application designed to stream live market data for 0 Days to Expiration (0DTE) S&P 500 Index (SPX) options using the Charles Schwab API. It integrates WebSocket streaming, REST API calls, and PostgreSQL storage to deliver a high-performance platform for capturing and analyzing real-time options data. The script dynamically adapts to SPX price movements, ensuring relevant option strikes are tracked, and persists data for downstream analysis. This README provides a comprehensive overview of the workflow, structure, advantages, and usage, showcasing the script's technical depth and programming expertise.

## Table of Contents
- [Project Overview](#project-overview)
- [Workflow](#workflow)
- [Project Structure](#project-structure)
- [Advantages](#advantages)
- [Usage](#usage)
- [Dependencies](#dependencies)
- [Logging and Monitoring](#logging-and-monitoring)
- [Contributing](#contributing)
- [License](#license)

## Project Overview
This script is tailored for traders, analysts, and developers needing real-time SPX 0DTE options data, which are highly volatile due to their same-day expiration. It leverages the Charles Schwab API for market data retrieval and WebSocket streaming, storing data in a PostgreSQL database for persistence and analysis. Key features include:

- Real-time streaming of Level 1 options data (bid, ask, delta, gamma, theta, vega, etc.).
- Dynamic subscription updates based on SPX price changes.
- Persistent storage in a PostgreSQL database with a structured schema.
- Fallback mechanisms for API failures and symbol generation.
- Comprehensive logging for debugging and monitoring.

## Workflow
The script follows a structured workflow to ensure seamless data streaming and storage. Below are the key steps, with code snippets to illustrate functionality.

1. **Database Initialization**  
   The script clears the PostgreSQL table to start with a fresh dataset:  
   ```python  
   def start_db():  
       '''Removes all records present in the db'''  
       conn = psycopg2.connect(**DB_CONFIG)  
       cur = conn.cursor()  
       cur.execute(f"TRUNCATE TABLE {TABLE_NAME} RESTART IDENTITY CASCADE")  
       conn.commit()  
       cur.close()  
       conn.close()  
   ```

2. **Authentication and Credentials**  
   It retrieves an OAuth token and streaming credentials from the Schwab API:  
   ```python  
   def get_access_token():  
       try:  
           with open(TOKEN_FILE, 'r') as f:  
               return json.load(f).get("access_token")  
       except Exception as e:  
           logger.error(f"Failed to read token file: {e}")  
           return None  

   def get_streamer_credentials():  
       try:  
           headers = {"Authorization": f"Bearer {get_access_token()}"}  
           response = requests.get(USER_PREF_URL, headers=headers)  
           if response.status_code != 200:  
               logger.error(f"Failed to get user preferences: {response.status_code}")  
               return None  
           data = response.json()  
           s_info = data["streamerInfo"][0]  
           return {  
               "url": s_info["streamerSocketUrl"],  
               "customerId": s_info["schwabClientCustomerId"],  
               "correlId": s_info["schwabClientCorrelId"]  
           }  
       except Exception as e:  
           logger.error(f"Failed to get streamer credentials: {e}")  
           return None  
   ```

3. **SPX Price Retrieval**  
   The script fetches the current SPX price from Schwab API, with fallbacks for reliability:  
   ```python  
   def get_spx_price():  
       try:  
           headers = {"Authorization": f"Bearer {get_access_token()}"}  
           response = requests.get(QUOTE_URL, headers=headers, params={"symbols": SPX_SYMBOL})  
           if response.status_code != 200:  
               for symbol in ["$SPX.X", "SPX", ".SPX"]:  
                   response = requests.get(QUOTE_URL, headers=headers, params={"symbols": symbol})  
                   if response.status_code == 200:  
                       break  
           if response.status_code != 200:  
               logger.error(f"Failed to get SPX price: {response.status_code}")  
               return 5440  
           data = response.json()  
           price = None  
           for key in [SPX_SYMBOL, "$SPX.X", "SPX", ".SPX"]:  
               if key in data and isinstance(data[key], dict):  
                   if "quote" in data[key] and "lastPrice" in data[key]["quote"]:  
                       price = data[key]["quote"]["lastPrice"]  
                       break  
                   elif "lastPrice" in data[key]:  
                       price = data[key]["lastPrice"]  
                       break  
           if price is None:  
               logger.warning("Unable to find SPX price in response, using default")  
               price = 5440  
           return round(float(price))  
       except Exception as e:  
           logger.error(f"Error getting SPX price: {e}")  
           return 5440  
   ```

4. **Option Symbol Generation**  
   It retrieves or generates 0DTE option symbols (~60 symbols) from schwabs API:  
   ```python  
   def get_spx_0dte_option_symbols():  
       try:  
           headers = {"Authorization": f"Bearer {get_access_token()}"}  
           for symbol in [SPX_SYMBOL, "$SPX.X", "SPX"]:  
               params = {  
                   "symbol": symbol,  
                   "contractType": "ALL",  
                   "includeQuotes": "FALSE",  
                   "fromDate": datetime.datetime.now().strftime('%Y-%m-%d'),  
                   "toDate": datetime.datetime.now().strftime('%Y-%m-%d'),  
                   "strikeCount": "60"  
               }  
               response = requests.get(CHAIN_URL, headers=headers, params=params)  
               if response.status_code == 200:  
                   chain_data = response.json()  
                   if "callExpDateMap" in chain_data or "putExpDateMap" in chain_data:  
                       logger.info(f"Got valid chain data using symbol: {symbol}")  
                       today = datetime.datetime.now().strftime('%Y-%m-%d')  
                       symbols = []  
                       for map_name in ["callExpDateMap", "putExpDateMap"]:  
                           exp_map = chain_data.get(map_name, {})  
                           for exp_date, strikes in exp_map.items():  
                               if today in exp_date.split(':')[0]:  
                                   for strike, contracts in strikes.items():  
                                       for contract in contracts:  
                                           if "symbol" in contract:  
                                               symbols.append(contract["symbol"])  
                       if symbols:  
                           logger.info(f"✅ Found {len(symbols)} option symbols from API")  
                           return symbols  
       except Exception as e:  
           logger.error(f"Error getting option chain: {e}")  
       spot = get_spx_price()  
       return generate_option_symbols(spot)  
   ```
    Resorts to manual symbol generation if the API fails. The `generate_option_symbols` function creates approximately 62 symbols (calls and puts) based on the SPX price:

    ```python
    def generate_option_symbols(spot_price):
    """
    Generate SPX option symbols for 0DTE options
    The spot_price is the output from get_spx_price()
    Outputs a list with ~ 62 option symbols
    """
    spot_rounded = round(spot_price / 5) * 5
    today = datetime.datetime.now().strftime('%y%m%d')
    strikes = list(range(spot_rounded - 75, spot_rounded + 80, 5))
       
    symbols = []
    for strike in strikes:
        strike_formatted = f"{strike * 1000:08d}"
        symbols.append(f"SPXW_{today}_C{strike_formatted}")
        symbols.append(f"SPXW_{today}_P{strike_formatted}")
    
    logger.info(f"Generated {len(symbols)} symbols. Sample: {symbols[0]}")
    return symbols
    ```

### 5. WebSocket Streaming
The core streaming logic in `run_streamer` establishes a WebSocket connection, logs in, and subscribes to the option symbols:

```python
async def run_streamer():
    start_db()
    current_spx_price = get_spx_price()
    creds = get_streamer_credentials()
    if not creds:
        logger.error("❌ Failed to get streaming credentials")
    symbols = get_spx_0dte_option_symbols()
    if not symbols:
        logger.error("❌ No symbols found. Will try again shortly.")
    async with websockets.connect(creds['url']) as ws:
        login = {
            "requests": [{
                "service": "ADMIN",
                "requestid": str(uuid.uuid4()),
                "command": "LOGIN",
                "SchwabClientCustomerId": creds["customerId"],
                "SchwabClientCorrelId": creds["correlId"],
                "parameters": {
                    "Authorization": get_access_token(),
                    "SchwabClientChannel": "N9",
                    "SchwabClientFunctionId": "APIAPP"
                }
            }]
        }
        await ws.send(json.dumps(login))
        login_resp = await ws.recv()
        login_data = json.loads(login_resp)
        login_content = login_data.get("response", [{}])[0].get("content", {})
        if login_content.get("code") != 0:
            logger.error(f"❌ Login failed: {login_content}")
        logger.info("✅ Login successful!")
        sub = {
            "requests": [{
                "service": STREAMING_SERVICE,
                "requestid": str(uuid.uuid4()),
                "command": "SUBS",
                "SchwabClientCustomerId": creds["customerId"],
                "SchwabClientCorrelId": creds["correlId"],
                "parameters": {
                    "keys": ",".join(symbols),
                    "fields": FIELDS
                }
            }]
        }
        await ws.send(json.dumps(sub))
        logger.info(f"📊 Subscribed {len(symbols)} symbols")
```

A keep-alive task ensures the WebSocket connection remains active:

```python
async def keep_alive(ws, creds):
    """Send heartbeat to keep connection alive"""
    while True:
        try:
            ping = {
                "requests": [{
                    "service": "ADMIN",
                    "requestid": str(uuid.uuid4()),
                    "command": "QOS",
                    "SchwabClientCustomerId": creds["customerId"],
                    "SchwabClientCorrelId": creds["correlId"],
                    "parameters": {"qoslevel": "0"}
                }]
            }
            await ws.send(json.dumps(ping))
            logger.info("❤️ Sent keepalive")
            await asyncio.sleep(30)
        except Exception as e:
            logger.error(f"Keepalive error: {e}")
            break
```

### 6. Dynamic Symbol Updates
The script monitors the SPX price and adjusts subscriptions if it changes significantly:

```python
spx_price = get_spx_price()
if (current_spx_price != spx_price) and (not first_time):
    logger.info(f"🔄 Underlying price moved from {current_spx_price} to {spx_price}")
    new_symbols = get_spx_0dte_option_symbols()
    strike_prices_to_delete = [item for item in symbols if item not in new_symbols]
    new_strike_prices_to_sub = [item for item in new_symbols if item not in symbols]
    symbols = new_symbols
    if len(strike_prices_to_delete) >= 1:
        unsub = {
            "requests": [{
                "service": STREAMING_SERVICE,
                "requestid": str(uuid.uuid4()),
                "command": "UNSUBS",
                "SchwabClientCustomerId": creds["customerId"],
                "SchwabClientCorrelId": creds["correlId"],
                "parameters": {
                    "keys": ",".join(strike_prices_to_delete),
                    "fields": FIELDS
                }
            }]
        }
        await ws.send(json.dumps(unsub))
        logger.info(f"📊 Unsubscribed from {len(strike_prices_to_delete)} symbols")
        remove_strikes_from_db(strike_prices_to_delete)
    if len(new_strike_prices_to_sub) >= 1:
        sub = {
            "requests": [{
                "service": STREAMING_SERVICE,
                "requestid": str(uuid.uuid4()),
                "command": "ADD",
                "SchwabClientCustomerId": creds["customerId"],
                "SchwabClientCorrelId": creds["correlId"],
                "parameters": {
                    "keys": ",".join(new_strike_prices_to_sub),
                    "fields": FIELDS
                }
            }]
        }
        await ws.send(json.dumps(sub))
        logger.info(f"📊 Subscribed to {len(new_strike_prices_to_sub)} symbols")
    current_spx_price = spx_price
```

### 7. Data Storage
Streaming data is parsed and stored in the PostgreSQL database using `insert_quote_to_db`:

```python
def insert_quote_to_db(symbol, quote_data):
    try:
        if isinstance(quote_data, list):
            for item in quote_data:
                if isinstance(item, dict) and item.get('key') == symbol:
                    _insert_single_quote(symbol, item)
        elif isinstance(quote_data, dict):
            _insert_single_quote(symbol, quote_data)
        else:
            logger.warning(f"Unexpected quote data type: {type(quote_data)}")
    except Exception as e:
        logger.error(f"Database error: {e}")
```

The `_insert_single_quote` function dynamically updates or inserts records, handling both call and put options:

```python
def _insert_single_quote(symbol, data):
    parts = symbol.strip().split()[-1]
    option_type = "call" if "C" in parts else "put"
    strike_price = int(parts[-8:]) / 1000
    try:
        conn = psycopg2.connect(**DB_CONFIG)
        cur = conn.cursor()
        cur.execute(f"SELECT 1 FROM {TABLE_NAME} WHERE strike_price = %s", (strike_price,))
        exists = cur.fetchone() is not None
        if exists:
            update_fields = []
            update_values = []
            for key, col in {
                "4": "last",
                "19": "net_chg",
                "28": "delta",
                "29": "gamma",
                "30": "theta",
                "9": "open_int",
                "10": "iv",
                "31": "vega",
                "2": "bid",
                "3": "ask",
                "20": "strike_price"
            }.items():
                if key in data:
                    update_fields.append(f"{col} = %s")
                    update_values.append(data[key])
            if update_fields and (strike_price != 0):
                if option_type == 'call':
                    update_fields = [f"call_{i}" if i not in ['strike_price = %s', 'exp_date = %s'] else i for i in update_fields]
                elif option_type == 'put':
                    update_fields = [f"put_{i}" if i not in ['strike_price = %s', 'exp_date = %s'] else i for i in update_fields]
                if ('12' in data) and ('23' in data) and ('26' in data):
                    if (data.get("12") != -1) and (data.get("23") != -1) and (data.get("26") != -1):
                        exp_date = f'{data.get("12")}-{data.get("23")}-{data.get("26")}'
                        update_fields.append(f"exp_date = %s")
                        update_values.append(exp_date)
                update_values.append(strike_price)
                try:
                    update_query = f"UPDATE {TABLE_NAME} SET {', '.join(update_fields)} WHERE strike_price = %s;"
                    cur.execute(update_query, tuple(update_values))
                    conn.commit()
                    cur.close()
                    conn.close()
                except:
                    pass
        else:
            if ('12' in data) and ('23' in data) and ('26' in data):
                if (data.get("12") != -1) and (data.get("23") != -1) and (data.get("26") != -1):
                    exp_date = f'{data.get("12")}-{data.get("23")}-{data.get("26")}'
                else:
                    exp_date = None
            else:
                exp_date = None
            insert_fields = ['net_chg', 'iv', 'open_int', 'vega', 'theta', 'gamma', 'delta', 'last', 'ask', 'bid', 'strike_price']
            insert_values = [data.get("19"), data.get("10"), data.get("9"), data.get("31"), data.get("30"), data.get("29"), data.get("28"), data.get("4"),
                             data.get("3"), data.get("2"), strike_price]
            if option_type == 'call':
                insert_fields = [f"call_{i}" if i not in ['strike_price', 'exp_date'] else i for i in insert_fields]
            elif option_type == 'put':
                insert_fields = [f"put_{i}" if i not in ['strike_price', 'exp_date'] else i for i in insert_fields]
            insert_fields.append('exp_date')
            insert_values.append(exp_date)
            if (strike_price != 0):
                insert_query = f"INSERT INTO {TABLE_NAME} ({', '.join(insert_fields)}) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s);"
                cur.execute(insert_query, tuple(insert_values))
                conn.commit()
                cur.close()
                conn.close()
        print(f"Successfully {'updated' if exists else 'inserted'} data for {symbol}")
    except Exception as e:
        print(f"Error inserting/updating quote for {symbol}: {e}")
```

### 8. Reconnection Logic
The `handle_reconnect` function ensures continuous operation by retrying connections on failure:

```python
async def handle_reconnect():
    while True:
        try:
            await run_streamer()
        except Exception as e:
            logger.error(f"Connection error: {e}")
        logger.info("🔄 Reconnecting in 5 seconds...")
        await asyncio.sleep(5)
```

## Project Structure
The project is organized for clarity and maintainability:
 ```plaintext
 spx-0dte-streamer/
 ├── spx_0dte_streamer.py    # Main application script
 ├── requirements.txt        # Python dependencies
 ├── /var/www/tokens/        # Directory for OAuth token file
 │   └── oauth_tokens.json
 └── README.md               # This documentation
 ```

## Advantages
The script offers several technical and practical benefits:
- **Dynamic Adaptation**: Automatically updates subscriptions based on SPX price movements, ensuring relevant data in volatile markets.
- **Robust Error Handling**: Implements fallbacks for API failures (e.g., alternate SPX symbols, default price of 5440) and reconnection logic for continuous operation.
- **Efficient Data Storage**: Uses a single PostgreSQL table with a prefix-based schema (e.g., `call_bid`, `put_ask`) for compact storage of call and put data.
- **Asynchronous Performance**: Leverages `asyncio` and `websockets` for non-blocking I/O, handling streaming, keep-alive pings, and price monitoring concurrently.
- **Comprehensive Logging**: Provides detailed, emoji-enhanced logs for connection status, subscription changes, and data updates, aiding debugging.
- **Extensibility**: Modular design allows easy addition of new indices, data fields, or analytics.

## Usage
### Prerequisites
1. **Python 3.8+**: Ensure Python is installed.
2. **PostgreSQL**: Set up a database named `trader_platform_data` with the following schema:
   ```python
   CREATE TABLE spx_0dte_stream (
       id SERIAL PRIMARY KEY,
       strike_price NUMERIC,
       exp_date DATE,
       call_last NUMERIC,
       call_bid NUMERIC,
       call_ask NUMERIC,
       call_net_chg NUMERIC,
       call_delta NUMERIC,
       call_gamma NUMERIC,
       call_theta NUMERIC,
       call_vega NUMERIC,
       call_iv NUMERIC,
       call_open_int NUMERIC,
       put_last NUMERIC,
       put_bid NUMERIC,
       put_ask NUMERIC,
       put_net_chg NUMERIC,
       put_delta NUMERIC,
       put_gamma NUMERIC,
       put_theta NUMERIC,
       put_vega NUMERIC,
       put_iv NUMERIC,
       put_open_int NUMERIC
   );
   ```
3. **Schwab API Access**: Obtain an OAuth token and store it in `/var/www/tokens/oauth_tokens.json`:
   ```python
   {
       "access_token": "your_access_token_here"
   }
   ```

### Installation
1. Clone the repository:
   ```python
   git clone https://github.com/Derrick-Mulwa/Real-Time-Options-Data-API-Streaming-Python-Script.git
   cd real-time-options-streamer
   ```
2. Install dependencies:
   ```python
   pip install -r requirements.txt
   ```
3. Configure the database in `spx_0dte_streamer.py` if needed:
   ```python
   DB_CONFIG = {"dbname": "trader_platform_data", "user": "trader", "password": "trader_pass", "host": "localhost"}
   ```

### Running the Script
Execute the script:
```python
python3 spx_0dte_streamer.py
```

The script will:
- Clear the database table.
- Fetch the SPX price and option symbols (~60 symbols: 30 calls, 30 puts).
- Establish a WebSocket connection to stream data and store it in the database.
- Monitor SPX price changes and dynamically adjust subscriptions.

To stop, press `Ctrl+C`:
```python
except KeyboardInterrupt:
    logger.info("👋 Stopped by user.")
```

### Monitoring
- **Logs**: Check console output for real-time updates:
  ```python
  2025-06-03 18:10:00,123 - ✅ Login successful!
  2025-06-03 18:10:00,124 - 📊 Subscribed 62 symbols
  2025-06-03 18:10:00,125 - 🔄 SPXW_250603C05570000 → bid:100.50, ask:101.00, delta:0.45, theta:-0.02
  ```
- **Database**: Query the `spx_0dte_stream` table to analyze data:
  ```python
  SELECT strike_price, call_bid, call_ask, put_bid, put_ask FROM spx_0dte_stream WHERE exp_date = CURRENT_DATE;
  ```

## Dependencies
The script requires the following Python packages, listed in `requirements.txt`:
```python
websockets==10.4
requests==2.31.0
psycopg2-binary==2.9.9
```

Install them using:
```python
pip install -r requirements.txt
```

These libraries provide:
- `websockets`: Asynchronous WebSocket communication.
- `requests`: HTTP requests for API calls.
- `psycopg2-binary`: PostgreSQL database connectivity.

## Logging and Monitoring
The script uses Python's `logging` module with an `INFO` level and a clear format:
```python
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(message)s')
logger = logging.getLogger("SPX_Streamer")
```

Logs cover:
- Connection events (e.g., login success/failure, WebSocket connection).
- Subscription changes (e.g., new strikes added, old strikes removed).
- Data updates (e.g., bid, ask, delta for each symbol).
- Errors (e.g., API failures, database issues).
- Heartbeats to confirm WebSocket health.

Logs are output to the console but can be redirected to a file by modifying the logging configuration.

## Conclusion
The Real-Time Options Data API Streaming Python Script is a powerful tool for capturing and analyzing SPX 0DTE options data, offering real-time insights into one of the most dynamic segments of the financial markets. Its robust design, with dynamic symbol management, efficient data storage, and reliable error handling, makes it ideal for traders seeking to capitalize on short-term market movements, analysts conducting high-frequency data studies, or developers building automated trading systems. The script’s asynchronous architecture ensures high performance, while its modular structure invites further customization, such as integrating additional indices or advanced analytics. By providing a seamless pipeline from data streaming to storage, this script empowers users to make data-driven decisions with confidence, positioning it as a valuable asset in any financial technology toolkit.