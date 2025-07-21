# Hyperliquid Python SDK Documentation

## Overview

The **Hyperliquid Python SDK** is the official Python client for interacting with the Hyperliquid decentralized exchange (DEX). It provides a comprehensive interface for trading, market data retrieval, account management, and real-time WebSocket connections.

## Installation

```bash
pip install hyperliquid-python-sdk
```

### Development Setup

For development environments, especially for Python 3.10:

```bash
brew install python@3.10 && poetry env use /opt/homebrew/Cellar/python@3.10/3.10.16/bin/python3.10
make install
```

## Core Components

### 1. Info Client - Market Data & Account Information

The Info client is used for retrieving public market data and account information without requiring authentication.

```python
from hyperliquid.info import Info
from hyperliquid.utils import constants

# Initialize Info client
info = Info(constants.TESTNET_API_URL, skip_ws=True)

# Get user state
user_state = info.user_state("0xcd5051944f780a621ee62e39e493c489668acf4d")
print(user_state)

# Get all mids (mid prices)
all_mids = info.all_mids()
print(all_mids)

# Get L2 order book
l2_book = info.l2_book("BTC")
print(l2_book)

# Get user's open orders
open_orders = info.open_orders("0x...")
print(open_orders)

# Get user fills (trade history)
user_fills = info.user_fills("0x...")
print(user_fills)

# Get candlestick data
candles = info.candle_snapshot({
    "coin": "BTC",
    "interval": "1h",
    "startTime": 1640995200000,
    "endTime": 1641081600000
})
print(candles)

# Get funding rate history
funding_history = info.funding_history("BTC", startTime=1640995200000, endTime=1641081600000)
print(funding_history)

# Get perpetuals metadata
meta = info.meta()
print(meta)

# Get spot metadata
spot_meta = info.spot_meta()
print(spot_meta)
```

### 2. Exchange Client - Trading Operations

The Exchange client handles all trading operations and requires authentication.

```python
from hyperliquid.exchange import Exchange
from hyperliquid.info import Info
from hyperliquid.utils import constants

# Initialize with private key
exchange = Exchange(config, base_url=constants.TESTNET_API_URL)

# Place a limit order
order_result = exchange.place_order({
    "coin": "BTC",
    "is_buy": True,
    "sz": 0.1,
    "limit_px": 30000,
    "order_type": {"limit": {"tif": "Gtc"}},
    "reduce_only": False
})
print(order_result)

# Place a market order
market_order = exchange.place_order({
    "coin": "ETH",
    "is_buy": False,
    "sz": 1.0,
    "limit_px": 2000,  # For market orders, use current market price
    "order_type": {"market": {}},
    "reduce_only": False
})
print(market_order)

# Cancel an order
cancel_result = exchange.cancel_order({
    "coin": "BTC",
    "o": 123456  # order ID
})
print(cancel_result)

# Cancel all orders
cancel_all = exchange.cancel_all_orders()
print(cancel_all)

# Cancel orders for specific coin
cancel_coin = exchange.cancel_all_orders("BTC")
print(cancel_coin)

# Modify an existing order
modify_result = exchange.modify_order({
    "oid": 123456,
    "order": {
        "coin": "BTC",
        "is_buy": True,
        "sz": 0.2,  # New size
        "limit_px": 31000,  # New price
        "order_type": {"limit": {"tif": "Gtc"}},
        "reduce_only": False
    }
})
print(modify_result)

# Transfer between spot and perpetual
transfer_result = exchange.transfer_between_spot_and_perp(100, True)  # 100 USDC from spot to perp
print(transfer_result)

# Withdraw USDC
withdraw_result = exchange.withdraw_usdc(50, "0x...")  # 50 USDC to address
print(withdraw_result)

# Update leverage
leverage_result = exchange.update_leverage("BTC", 10)  # Set 10x leverage for BTC
print(leverage_result)

# Update isolated margin
margin_result = exchange.update_isolated_margin("BTC", True)  # Enable isolated margin for BTC
print(margin_result)
```

### 3. WebSocket Client - Real-time Data

For real-time market data and user updates:

```python
from hyperliquid.info import Info
from hyperliquid.utils import constants
import asyncio

# Initialize with WebSocket enabled
info = Info(constants.MAINNET_API_URL, skip_ws=False)

# Subscribe to all mids
def on_all_mids(data):
    print(f"All mids update: {data}")

info.subscribe({"type": "allMids"}, on_all_mids)

# Subscribe to L2 book updates
def on_l2_book(data):
    print(f"L2 book update for {data['coin']}: {data}")

info.subscribe({"type": "l2Book", "coin": "BTC"}, on_l2_book)

# Subscribe to user fills
def on_user_fills(data):
    print(f"User fill: {data}")

info.subscribe({
    "type": "userFills", 
    "user": "0x..."
}, on_user_fills)

# Subscribe to user events
def on_user_events(data):
    print(f"User event: {data}")

info.subscribe({
    "type": "userEvents", 
    "user": "0x..."
}, on_user_events)

# Subscribe to candle updates
def on_candle(data):
    print(f"Candle update: {data}")

info.subscribe({
    "type": "candle",
    "coin": "ETH",
    "interval": "1m"
}, on_candle)

# Keep the connection alive
asyncio.get_event_loop().run_forever()
```

## Configuration

### Basic Configuration

```python
# config.json example
{
    "mainnet_config": {
        "api_url": "https://api.hyperliquid.xyz",
        "private_key": "0x...",
        "account_address": "0x..."
    },
    "testnet_config": {
        "api_url": "https://api.hyperliquid-testnet.xyz",
        "private_key": "0x...",
        "account_address": "0x..."
    }
}
```

### Environment Variables

```python
import os
from hyperliquid.info import Info
from hyperliquid.exchange import Exchange

# Using environment variables
private_key = os.getenv('HYPERLIQUID_PRIVATE_KEY')
api_url = os.getenv('HYPERLIQUID_API_URL', 'https://api.hyperliquid.xyz')

# Initialize clients
info = Info(api_url, skip_ws=True)
exchange = Exchange({
    'private_key': private_key,
    'account_address': '0x...'  # Derived from private key
}, base_url=api_url)
```

## Advanced Features

### 1. Batch Operations

```python
# Place multiple orders at once
batch_orders = exchange.place_order([
    {
        "coin": "BTC",
        "is_buy": True,
        "sz": 0.1,
        "limit_px": 30000,
        "order_type": {"limit": {"tif": "Gtc"}},
        "reduce_only": False
    },
    {
        "coin": "ETH",
        "is_buy": False,
        "sz": 1.0,
        "limit_px": 2000,
        "order_type": {"limit": {"tif": "Gtc"}},
        "reduce_only": False
    }
])
print(batch_orders)

# Cancel multiple orders
cancel_batch = exchange.cancel_order([
    {"coin": "BTC", "o": 123456},
    {"coin": "ETH", "o": 789012}
])
print(cancel_batch)
```

### 2. Advanced Order Types

```python
# Stop-loss order
stop_loss = exchange.place_order({
    "coin": "BTC",
    "is_buy": False,
    "sz": 0.1,
    "limit_px": 29000,
    "order_type": {
        "trigger": {
            "trigger_px": 29500,
            "is_market": True,
            "tpsl": "sl"  # stop-loss
        }
    },
    "reduce_only": True
})

# Take-profit order
take_profit = exchange.place_order({
    "coin": "BTC",
    "is_buy": False,
    "sz": 0.1,
    "limit_px": 35000,
    "order_type": {
        "trigger": {
            "trigger_px": 34500,
            "is_market": False,
            "tpsl": "tp"  # take-profit
        }
    },
    "reduce_only": True
})

# Good Till Time (GTT) order
gtt_order = exchange.place_order({
    "coin": "ETH",
    "is_buy": True,
    "sz": 1.0,
    "limit_px": 1800,
    "order_type": {
        "limit": {
            "tif": "Gtt",
            "good_till": 1640995200000  # Unix timestamp
        }
    },
    "reduce_only": False
})
```

### 3. Portfolio Management

```python
# Get account summary
account_summary = info.user_state("0x...")
print(f"Account value: {account_summary['marginSummary']['accountValue']}")
print(f"Total margin used: {account_summary['marginSummary']['totalMarginUsed']}")

# Get positions
positions = account_summary.get('assetPositions', [])
for position in positions:
    if float(position['position']['szi']) != 0:
        print(f"Position in {position['position']['coin']}: {position['position']['szi']}")

# Get balances
balances = account_summary.get('withdrawable', 0)
print(f"Withdrawable balance: {balances}")

# Risk management
def check_risk_limits(user_address):
    state = info.user_state(user_address)
    margin_summary = state['marginSummary']
    
    account_value = float(margin_summary['accountValue'])
    total_margin_used = float(margin_summary['totalMarginUsed'])
    
    if account_value > 0:
        margin_ratio = total_margin_used / account_value
        print(f"Margin utilization: {margin_ratio:.2%}")
        
        if margin_ratio > 0.8:
            print("⚠️  HIGH RISK: Margin utilization above 80%")
            return False
        elif margin_ratio > 0.6:
            print("⚠️  MEDIUM RISK: Margin utilization above 60%")
            return True
        else:
            print("✅ LOW RISK: Margin utilization below 60%")
            return True
    return False
```

### 4. Market Analysis Tools

```python
import pandas as pd
from datetime import datetime, timedelta

def analyze_market_data(coin, days=7):
    """Analyze market data for a given coin"""
    end_time = int(datetime.now().timestamp() * 1000)
    start_time = int((datetime.now() - timedelta(days=days)).timestamp() * 1000)
    
    # Get candlestick data
    candles = info.candle_snapshot({
        "coin": coin,
        "interval": "1h",
        "startTime": start_time,
        "endTime": end_time
    })
    
    if not candles:
        return None
    
    # Convert to DataFrame
    df = pd.DataFrame(candles)
    df['timestamp'] = pd.to_datetime(df['t'], unit='ms')
    df['open'] = df['o'].astype(float)
    df['high'] = df['h'].astype(float)
    df['low'] = df['l'].astype(float)
    df['close'] = df['c'].astype(float)
    df['volume'] = df['v'].astype(float)
    
    # Calculate technical indicators
    df['sma_20'] = df['close'].rolling(window=20).mean()
    df['sma_50'] = df['close'].rolling(window=50).mean()
    df['volatility'] = df['close'].pct_change().rolling(window=24).std()
    
    # Current metrics
    current_price = df['close'].iloc[-1]
    sma_20 = df['sma_20'].iloc[-1]
    sma_50 = df['sma_50'].iloc[-1]
    volatility = df['volatility'].iloc[-1]
    
    print(f"Market Analysis for {coin}")
    print(f"Current Price: ${current_price:.2f}")
    print(f"20-hour SMA: ${sma_20:.2f}")
    print(f"50-hour SMA: ${sma_50:.2f}")
    print(f"24h Volatility: {volatility:.4f}")
    
    # Trend analysis
    if current_price > sma_20 > sma_50:
        print("📈 Bullish trend (Price > SMA20 > SMA50)")
    elif current_price < sma_20 < sma_50:
        print("📉 Bearish trend (Price < SMA20 < SMA50)")
    else:
        print("📊 Sideways/Mixed trend")
    
    return df

# Usage
btc_data = analyze_market_data("BTC", days=7)
```

## Error Handling & Best Practices

### Error Handling

```python
from hyperliquid.info import Info
from hyperliquid.exchange import Exchange
import time
import logging

# Set up logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_place_order(exchange, order_params, max_retries=3):
    """Place order with retry logic"""
    for attempt in range(max_retries):
        try:
            result = exchange.place_order(order_params)
            if result.get('status') == 'ok':
                logger.info(f"Order placed successfully: {result}")
                return result
            else:
                logger.warning(f"Order failed: {result}")
                return result
        except Exception as e:
            logger.error(f"Attempt {attempt + 1} failed: {str(e)}")
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise e

def safe_get_user_state(info, user_address, max_retries=3):
    """Get user state with retry logic"""
    for attempt in range(max_retries):
        try:
            return info.user_state(user_address)
        except Exception as e:
            logger.error(f"Failed to get user state (attempt {attempt + 1}): {str(e)}")
            if attempt < max_retries - 1:
                time.sleep(1)
            else:
                raise e
```

### Rate Limiting

```python
import time
from functools import wraps

def rate_limit(calls_per_second=10):
    """Rate limiting decorator"""
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            if left_to_wait > 0:
                time.sleep(left_to_wait)
            ret = func(*args, **kwargs)
            last_called[0] = time.time()
            return ret
        return wrapper
    return decorator

# Usage
@rate_limit(calls_per_second=5)
def get_market_data(coin):
    return info.l2_book(coin)
```

## Testing & Development

### Running Tests

```bash
# Install dependencies
make install

# Run tests
make test

# Run linting
make lint

# Run safety checks
make check-safety

# Update dependencies
make update-dev-deps
```

### Example Trading Bot

```python
import asyncio
import logging
from hyperliquid.info import Info
from hyperliquid.exchange import Exchange
from hyperliquid.utils import constants

class SimpleGridBot:
    def __init__(self, config):
        self.info = Info(config['api_url'], skip_ws=False)
        self.exchange = Exchange(config, base_url=config['api_url'])
        self.coin = config['coin']
        self.grid_size = config['grid_size']
        self.order_size = config['order_size']
        self.active_orders = {}
        
    async def start(self):
        """Start the grid trading bot"""
        logging.info(f"Starting grid bot for {self.coin}")
        
        # Get current price
        all_mids = self.info.all_mids()
        current_price = float(all_mids[self.coin])
        
        # Place initial grid orders
        await self.place_grid_orders(current_price)
        
        # Subscribe to fills
        self.info.subscribe({
            "type": "userFills",
            "user": self.exchange.account_address
        }, self.on_fill)
        
        # Keep running
        while True:
            await asyncio.sleep(1)
    
    async def place_grid_orders(self, center_price):
        """Place buy and sell orders around center price"""
        # Place buy orders below current price
        for i in range(1, self.grid_size + 1):
            buy_price = center_price * (1 - 0.01 * i)  # 1% intervals
            await self.place_order(buy_price, True)
        
        # Place sell orders above current price
        for i in range(1, self.grid_size + 1):
            sell_price = center_price * (1 + 0.01 * i)  # 1% intervals
            await self.place_order(sell_price, False)
    
    async def place_order(self, price, is_buy):
        """Place a single grid order"""
        try:
            result = self.exchange.place_order({
                "coin": self.coin,
                "is_buy": is_buy,
                "sz": self.order_size,
                "limit_px": price,
                "order_type": {"limit": {"tif": "Gtc"}},
                "reduce_only": False
            })
            
            if result.get('status') == 'ok':
                order_id = result['response']['data']['statuses'][0]['resting']['oid']
                self.active_orders[order_id] = {
                    'price': price,
                    'is_buy': is_buy,
                    'size': self.order_size
                }
                logging.info(f"Placed {'buy' if is_buy else 'sell'} order at ${price:.2f}")
        except Exception as e:
            logging.error(f"Failed to place order: {str(e)}")
    
    def on_fill(self, fill_data):
        """Handle order fills"""
        for fill in fill_data.get('fills', []):
            order_id = fill.get('oid')
            if order_id in self.active_orders:
                order = self.active_orders[order_id]
                logging.info(f"Order filled: {'buy' if order['is_buy'] else 'sell'} at ${order['price']:.2f}")
                
                # Place opposite order
                opposite_price = order['price'] * (1.02 if order['is_buy'] else 0.98)
                asyncio.create_task(self.place_order(opposite_price, not order['is_buy']))
                
                # Remove filled order
                del self.active_orders[order_id]

# Configuration
config = {
    'api_url': constants.TESTNET_API_URL,
    'private_key': 'your_private_key',
    'account_address': 'your_account_address',
    'coin': 'BTC',
    'grid_size': 5,
    'order_size': 0.01
}

# Run bot
bot = SimpleGridBot(config)
asyncio.run(bot.start())
```

This comprehensive documentation covers all aspects of the Hyperliquid Python SDK, from basic usage to advanced trading strategies and bot development. The SDK provides powerful tools for both retail and institutional traders to interact with the Hyperliquid DEX programmatically. 