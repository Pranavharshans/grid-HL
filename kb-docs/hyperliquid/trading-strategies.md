# Hyperliquid Trading Strategies Documentation

## Overview

This documentation provides comprehensive examples of trading strategies that can be implemented using the Hyperliquid platform. It covers various algorithmic trading approaches, risk management techniques, and practical implementation examples using multiple programming languages.

## Table of Contents

1. [Market Making Strategies](#market-making-strategies)
2. [Arbitrage Strategies](#arbitrage-strategies)
3. [Momentum Trading](#momentum-trading)
4. [Mean Reversion](#mean-reversion)
5. [Grid Trading](#grid-trading)
6. [DCA Strategies](#dca-strategies)
7. [Risk Management](#risk-management)
8. [Portfolio Management](#portfolio-management)

## Market Making Strategies

### Basic Market Making

Market making involves placing both buy and sell orders around the current market price to capture the bid-ask spread.

#### Python Implementation

```python
import asyncio
import logging
from hyperliquid.info import Info
from hyperliquid.exchange import Exchange
from hyperliquid.utils import constants

class MarketMaker:
    def __init__(self, config):
        self.info = Info(config['api_url'], skip_ws=False)
        self.exchange = Exchange(config, base_url=config['api_url'])
        self.symbol = config['symbol']
        self.spread = config['spread']  # 0.001 = 0.1%
        self.order_size = config['order_size']
        self.max_position = config['max_position']
        self.active_orders = {}
        
    async def start(self):
        """Start the market making strategy"""
        logging.info(f"Starting market maker for {self.symbol}")
        
        # Subscribe to price updates
        self.info.subscribe({"type": "allMids"}, self.on_price_update)
        
        # Subscribe to order fills
        self.info.subscribe({
            "type": "userFills",
            "user": self.exchange.account_address
        }, self.on_fill)
        
        # Initial order placement
        await self.update_orders()
        
        # Keep running
        while True:
            await asyncio.sleep(1)
    
    async def on_price_update(self, data):
        """Handle price updates"""
        if self.symbol in data:
            current_price = float(data[self.symbol])
            await self.update_orders(current_price)
    
    async def update_orders(self, current_price=None):
        """Update buy and sell orders around current price"""
        if current_price is None:
            all_mids = self.info.all_mids()
            current_price = float(all_mids[self.symbol])
        
        # Calculate bid and ask prices
        bid_price = current_price * (1 - self.spread / 2)
        ask_price = current_price * (1 + self.spread / 2)
        
        # Cancel existing orders
        await self.cancel_all_orders()
        
        # Check current position
        user_state = self.info.user_state(self.exchange.account_address)
        current_position = self.get_current_position(user_state)
        
        # Place new orders if within position limits
        if abs(current_position + self.order_size) <= self.max_position:
            await self.place_buy_order(bid_price)
        
        if abs(current_position - self.order_size) <= self.max_position:
            await self.place_sell_order(ask_price)
    
    async def place_buy_order(self, price):
        """Place buy order"""
        try:
            result = self.exchange.place_order({
                "coin": self.symbol,
                "is_buy": True,
                "sz": self.order_size,
                "limit_px": price,
                "order_type": {"limit": {"tif": "Gtc"}},
                "reduce_only": False
            })
            
            if result.get('status') == 'ok':
                order_id = result['response']['data']['statuses'][0]['resting']['oid']
                self.active_orders[order_id] = {'side': 'buy', 'price': price}
                logging.info(f"Buy order placed at ${price:.2f}")
        except Exception as e:
            logging.error(f"Failed to place buy order: {str(e)}")
    
    async def place_sell_order(self, price):
        """Place sell order"""
        try:
            result = self.exchange.place_order({
                "coin": self.symbol,
                "is_buy": False,
                "sz": self.order_size,
                "limit_px": price,
                "order_type": {"limit": {"tif": "Gtc"}},
                "reduce_only": False
            })
            
            if result.get('status') == 'ok':
                order_id = result['response']['data']['statuses'][0]['resting']['oid']
                self.active_orders[order_id] = {'side': 'sell', 'price': price}
                logging.info(f"Sell order placed at ${price:.2f}")
        except Exception as e:
            logging.error(f"Failed to place sell order: {str(e)}")
    
    def on_fill(self, fill_data):
        """Handle order fills"""
        for fill in fill_data.get('fills', []):
            order_id = fill.get('oid')
            if order_id in self.active_orders:
                order = self.active_orders[order_id]
                logging.info(f"Order filled: {order['side']} at ${order['price']:.2f}")
                del self.active_orders[order_id]
                
                # Immediately try to place a new order on the opposite side
                asyncio.create_task(self.update_orders())

# Configuration
config = {
    'api_url': constants.TESTNET_API_URL,
    'private_key': 'your_private_key',
    'account_address': 'your_account_address',
    'symbol': 'BTC',
    'spread': 0.002,  # 0.2%
    'order_size': 0.01,
    'max_position': 0.1
}

# Run market maker
maker = MarketMaker(config)
asyncio.run(maker.start())
```

#### TypeScript Implementation

```typescript
import * as hl from "@nktkas/hyperliquid";

class MarketMaker {
  private infoClient: hl.InfoClient;
  private exchClient: hl.ExchangeClient;
  private subsClient: hl.SubscriptionClient;
  private symbol: string;
  private spread: number;
  private orderSize: string;
  private maxPosition: number;
  private activeOrders = new Map();

  constructor(privateKey: string, config: any) {
    const httpTransport = new hl.HttpTransport({ isTestnet: config.isTestnet });
    const wsTransport = new hl.WebSocketTransport({ 
      url: config.wsUrl 
    });
    
    this.infoClient = new hl.InfoClient({ transport: httpTransport });
    this.exchClient = new hl.ExchangeClient({ 
      wallet: privateKey, 
      transport: httpTransport,
      isTestnet: config.isTestnet 
    });
    this.subsClient = new hl.SubscriptionClient({ transport: wsTransport });
    
    this.symbol = config.symbol;
    this.spread = config.spread;
    this.orderSize = config.orderSize;
    this.maxPosition = config.maxPosition;
  }

  async start() {
    console.log(`Starting market maker for ${this.symbol}`);
    
    // Subscribe to price updates
    await this.subsClient.allMids((data) => {
      this.onPriceUpdate(data);
    });
    
    // Subscribe to order updates
    await this.subsClient.orderUpdates({ user: await this.getAddress() }, (data) => {
      this.onOrderUpdate(data);
    });
    
    // Initial order placement
    await this.updateOrders();
  }

  private async onPriceUpdate(data: any) {
    if (this.symbol in data) {
      const currentPrice = parseFloat(data[this.symbol]);
      await this.updateOrders(currentPrice);
    }
  }

  private async updateOrders(currentPrice?: number) {
    if (!currentPrice) {
      const allMids = await this.infoClient.allMids();
      currentPrice = parseFloat((allMids as any)[this.symbol]);
    }

    const bidPrice = currentPrice * (1 - this.spread / 2);
    const askPrice = currentPrice * (1 + this.spread / 2);

    // Cancel existing orders
    await this.cancelAllOrders();

    // Check current position
    const clearinghouseState = await this.infoClient.clearinghouseState({ 
      user: await this.getAddress() 
    });
    
    const currentPosition = this.getCurrentPosition(clearinghouseState);

    // Place new orders if within position limits
    if (Math.abs(currentPosition + parseFloat(this.orderSize)) <= this.maxPosition) {
      await this.placeBuyOrder(bidPrice);
    }

    if (Math.abs(currentPosition - parseFloat(this.orderSize)) <= this.maxPosition) {
      await this.placeSellOrder(askPrice);
    }
  }

  private async placeBuyOrder(price: number) {
    try {
      const result = await this.exchClient.order({
        orders: [{
          a: this.getAssetIndex(this.symbol),
          b: true,
          p: price.toFixed(4),
          s: this.orderSize,
          r: false,
          t: { limit: { tif: "Gtc" } }
        }],
        grouping: "na"
      });

      console.log(`Buy order placed at $${price.toFixed(2)}`);
    } catch (error) {
      console.error(`Failed to place buy order: ${error}`);
    }
  }

  private async placeSellOrder(price: number) {
    try {
      const result = await this.exchClient.order({
        orders: [{
          a: this.getAssetIndex(this.symbol),
          b: false,
          p: price.toFixed(4),
          s: this.orderSize,
          r: false,
          t: { limit: { tif: "Gtc" } }
        }],
        grouping: "na"
      });

      console.log(`Sell order placed at $${price.toFixed(2)}`);
    } catch (error) {
      console.error(`Failed to place sell order: ${error}`);
    }
  }

  private getAssetIndex(symbol: string): number {
    const assetMap: { [key: string]: number } = {
      "BTC": 0, "ETH": 1, "SOL": 2
    };
    return assetMap[symbol] || 0;
  }
}
```

### Advanced Market Making with Inventory Management

```python
class AdvancedMarketMaker(MarketMaker):
    def __init__(self, config):
        super().__init__(config)
        self.inventory_target = config.get('inventory_target', 0)
        self.inventory_skew_factor = config.get('inventory_skew_factor', 0.1)
        self.volatility_window = config.get('volatility_window', 100)
        self.price_history = []
        
    async def calculate_volatility(self):
        """Calculate recent price volatility"""
        if len(self.price_history) < self.volatility_window:
            return 0.02  # Default volatility
        
        returns = []
        for i in range(1, len(self.price_history)):
            ret = (self.price_history[i] / self.price_history[i-1]) - 1
            returns.append(ret)
        
        import numpy as np
        return np.std(returns) * np.sqrt(8760)  # Annualized volatility
    
    async def update_orders(self, current_price=None):
        """Update orders with inventory management"""
        if current_price is None:
            all_mids = self.info.all_mids()
            current_price = float(all_mids[self.symbol])
        
        # Update price history
        self.price_history.append(current_price)
        if len(self.price_history) > self.volatility_window:
            self.price_history.pop(0)
        
        # Calculate volatility and adjust spread
        volatility = await self.calculate_volatility()
        dynamic_spread = max(self.spread, volatility * 0.5)
        
        # Get current position and calculate inventory skew
        user_state = self.info.user_state(self.exchange.account_address)
        current_position = self.get_current_position(user_state)
        inventory_skew = (current_position - self.inventory_target) * self.inventory_skew_factor
        
        # Adjust prices based on inventory
        bid_price = current_price * (1 - dynamic_spread / 2 - inventory_skew)
        ask_price = current_price * (1 + dynamic_spread / 2 - inventory_skew)
        
        # Adjust order sizes based on inventory
        buy_size = self.order_size * (1 + inventory_skew) if current_position < self.inventory_target else self.order_size * 0.5
        sell_size = self.order_size * (1 - inventory_skew) if current_position > self.inventory_target else self.order_size * 0.5
        
        # Cancel existing orders
        await self.cancel_all_orders()
        
        # Place new orders
        if abs(current_position + buy_size) <= self.max_position:
            await self.place_buy_order(bid_price, buy_size)
        
        if abs(current_position - sell_size) <= self.max_position:
            await self.place_sell_order(ask_price, sell_size)
```

## Grid Trading Strategies

### Basic Grid Trading

Grid trading places buy and sell orders at regular intervals above and below the current price.

```python
class GridTrader:
    def __init__(self, config):
        self.info = Info(config['api_url'], skip_ws=False)
        self.exchange = Exchange(config, base_url=config['api_url'])
        self.symbol = config['symbol']
        self.grid_size = config['grid_size']  # Price difference between levels
        self.order_size = config['order_size']
        self.num_levels = config['num_levels']  # Number of levels on each side
        self.grid_orders = {}
        
    async def initialize_grid(self):
        """Initialize the grid around current price"""
        all_mids = self.info.all_mids()
        center_price = float(all_mids[self.symbol])
        
        # Place buy orders below current price
        for i in range(1, self.num_levels + 1):
            price = center_price - (self.grid_size * i)
            await self.place_grid_order('buy', price, i)
        
        # Place sell orders above current price
        for i in range(1, self.num_levels + 1):
            price = center_price + (self.grid_size * i)
            await self.place_grid_order('sell', price, i)
    
    async def place_grid_order(self, side, price, level):
        """Place a grid order"""
        try:
            result = self.exchange.place_order({
                "coin": self.symbol,
                "is_buy": side == 'buy',
                "sz": self.order_size,
                "limit_px": price,
                "order_type": {"limit": {"tif": "Gtc"}},
                "reduce_only": False
            })
            
            if result.get('status') == 'ok':
                order_id = result['response']['data']['statuses'][0]['resting']['oid']
                self.grid_orders[order_id] = {
                    'side': side,
                    'price': price,
                    'level': level
                }
                logging.info(f"Grid {side} order placed at ${price:.2f} (level {level})")
        except Exception as e:
            logging.error(f"Failed to place grid order: {str(e)}")
    
    def on_fill(self, fill_data):
        """Handle grid order fills"""
        for fill in fill_data.get('fills', []):
            order_id = fill.get('oid')
            if order_id in self.grid_orders:
                order = self.grid_orders[order_id]
                logging.info(f"Grid order filled: {order['side']} at ${order['price']:.2f}")
                
                # Place opposite order at the next level
                if order['side'] == 'buy':
                    new_price = order['price'] + self.grid_size
                    asyncio.create_task(self.place_grid_order('sell', new_price, order['level']))
                else:
                    new_price = order['price'] - self.grid_size
                    asyncio.create_task(self.place_grid_order('buy', new_price, order['level']))
                
                del self.grid_orders[order_id]
```

### Dynamic Grid Trading

```python
class DynamicGridTrader(GridTrader):
    def __init__(self, config):
        super().__init__(config)
        self.volatility_lookback = config.get('volatility_lookback', 24)
        self.grid_adjustment_factor = config.get('grid_adjustment_factor', 2.0)
        
    async def calculate_dynamic_grid_size(self):
        """Calculate grid size based on recent volatility"""
        # Get recent candlestick data
        end_time = int(time.time() * 1000)
        start_time = end_time - (self.volatility_lookback * 3600 * 1000)  # Hours to milliseconds
        
        candles = self.info.candle_snapshot({
            "coin": self.symbol,
            "interval": "1h",
            "startTime": start_time,
            "endTime": end_time
        })
        
        if len(candles) < 2:
            return self.grid_size
        
        # Calculate average true range (ATR)
        atr_values = []
        for i in range(1, len(candles)):
            high = float(candles[i]['h'])
            low = float(candles[i]['l'])
            prev_close = float(candles[i-1]['c'])
            
            tr = max(
                high - low,
                abs(high - prev_close),
                abs(low - prev_close)
            )
            atr_values.append(tr)
        
        avg_atr = sum(atr_values) / len(atr_values)
        return avg_atr * self.grid_adjustment_factor
    
    async def rebalance_grid(self):
        """Rebalance grid based on current market conditions"""
        # Calculate new grid size
        new_grid_size = await self.calculate_dynamic_grid_size()
        
        if abs(new_grid_size - self.grid_size) / self.grid_size > 0.2:  # 20% change threshold
            logging.info(f"Rebalancing grid: {self.grid_size:.4f} -> {new_grid_size:.4f}")
            
            # Cancel all existing orders
            await self.cancel_all_orders()
            
            # Update grid size
            self.grid_size = new_grid_size
            
            # Reinitialize grid
            await self.initialize_grid()
```

## Momentum Trading

### Trend Following Strategy

```typescript
class MomentumTrader {
  private infoClient: hl.InfoClient;
  private exchClient: hl.ExchangeClient;
  private symbol: string;
  private lookbackPeriod: number;
  private momentumThreshold: number;
  private positionSize: string;
  private priceHistory: number[] = [];

  constructor(privateKey: string, config: any) {
    const httpTransport = new hl.HttpTransport({ isTestnet: config.isTestnet });
    
    this.infoClient = new hl.InfoClient({ transport: httpTransport });
    this.exchClient = new hl.ExchangeClient({ 
      wallet: privateKey, 
      transport: httpTransport,
      isTestnet: config.isTestnet 
    });
    
    this.symbol = config.symbol;
    this.lookbackPeriod = config.lookbackPeriod;
    this.momentumThreshold = config.momentumThreshold;
    this.positionSize = config.positionSize;
  }

  async start() {
    // Initialize price history
    await this.initializePriceHistory();
    
    // Subscribe to price updates
    const wsTransport = new hl.WebSocketTransport();
    const subsClient = new hl.SubscriptionClient({ transport: wsTransport });
    
    await subsClient.allMids((data) => {
      this.onPriceUpdate(data);
    });
  }

  private async initializePriceHistory() {
    const endTime = Date.now();
    const startTime = endTime - (this.lookbackPeriod * 60 * 60 * 1000); // Hours to ms
    
    const candles = await this.infoClient.candleSnapshot({
      coin: this.symbol,
      interval: "1h",
      startTime,
      endTime
    });

    this.priceHistory = candles.map(candle => parseFloat(candle.c));
  }

  private onPriceUpdate(data: any) {
    if (this.symbol in data) {
      const currentPrice = parseFloat(data[this.symbol]);
      this.priceHistory.push(currentPrice);
      
      // Keep only the required lookback period
      if (this.priceHistory.length > this.lookbackPeriod) {
        this.priceHistory.shift();
      }
      
      // Check for momentum signals
      this.checkMomentumSignal(currentPrice);
    }
  }

  private async checkMomentumSignal(currentPrice: number) {
    if (this.priceHistory.length < this.lookbackPeriod) return;

    // Calculate momentum (rate of change)
    const oldPrice = this.priceHistory[0];
    const momentum = (currentPrice - oldPrice) / oldPrice;

    // Get current position
    const clearinghouseState = await this.infoClient.clearinghouseState({ 
      user: await this.getAddress() 
    });
    
    const currentPosition = this.getCurrentPosition(clearinghouseState);

    // Generate signals
    if (momentum > this.momentumThreshold && currentPosition <= 0) {
      // Strong upward momentum - go long
      await this.openLongPosition(currentPrice);
    } else if (momentum < -this.momentumThreshold && currentPosition >= 0) {
      // Strong downward momentum - go short
      await this.openShortPosition(currentPrice);
    }
  }

  private async openLongPosition(price: number) {
    try {
      // Close any short position first
      const currentPosition = await this.getCurrentPosition();
      if (currentPosition < 0) {
        await this.closePosition();
      }

      // Open long position
      await this.exchClient.order({
        orders: [{
          a: this.getAssetIndex(this.symbol),
          b: true,
          p: (price * 1.001).toFixed(4), // Small slippage tolerance
          s: this.positionSize,
          r: false,
          t: { limit: { tif: "Ioc" } } // Immediate or cancel
        }],
        grouping: "na"
      });

      console.log(`Opened long position at $${price.toFixed(2)}`);
    } catch (error) {
      console.error(`Failed to open long position: ${error}`);
    }
  }

  private async openShortPosition(price: number) {
    try {
      // Close any long position first
      const currentPosition = await this.getCurrentPosition();
      if (currentPosition > 0) {
        await this.closePosition();
      }

      // Open short position
      await this.exchClient.order({
        orders: [{
          a: this.getAssetIndex(this.symbol),
          b: false,
          p: (price * 0.999).toFixed(4), // Small slippage tolerance
          s: this.positionSize,
          r: false,
          t: { limit: { tif: "Ioc" } } // Immediate or cancel
        }],
        grouping: "na"
      });

      console.log(`Opened short position at $${price.toFixed(2)}`);
    } catch (error) {
      console.error(`Failed to open short position: ${error}`);
    }
  }
}
```

## Mean Reversion Strategies

### Bollinger Bands Strategy

```python
import numpy as np
from collections import deque

class BollingerBandsStrategy:
    def __init__(self, config):
        self.info = Info(config['api_url'], skip_ws=False)
        self.exchange = Exchange(config, base_url=config['api_url'])
        self.symbol = config['symbol']
        self.period = config.get('period', 20)
        self.std_dev = config.get('std_dev', 2)
        self.position_size = config['position_size']
        self.price_history = deque(maxlen=self.period)
        
    async def start(self):
        """Start the Bollinger Bands strategy"""
        await self.initialize_price_history()
        
        # Subscribe to price updates
        self.info.subscribe({"type": "allMids"}, self.on_price_update)
        
        while True:
            await asyncio.sleep(1)
    
    async def initialize_price_history(self):
        """Initialize price history with recent data"""
        end_time = int(time.time() * 1000)
        start_time = end_time - (self.period * 3600 * 1000)  # Hours to milliseconds
        
        candles = self.info.candle_snapshot({
            "coin": self.symbol,
            "interval": "1h",
            "startTime": start_time,
            "endTime": end_time
        })
        
        for candle in candles[-self.period:]:
            self.price_history.append(float(candle['c']))
    
    def calculate_bollinger_bands(self):
        """Calculate Bollinger Bands"""
        if len(self.price_history) < self.period:
            return None, None, None
        
        prices = np.array(self.price_history)
        sma = np.mean(prices)
        std = np.std(prices)
        
        upper_band = sma + (self.std_dev * std)
        lower_band = sma - (self.std_dev * std)
        
        return upper_band, sma, lower_band
    
    async def on_price_update(self, data):
        """Handle price updates and generate signals"""
        if self.symbol not in data:
            return
        
        current_price = float(data[self.symbol])
        self.price_history.append(current_price)
        
        # Calculate Bollinger Bands
        upper_band, middle_band, lower_band = self.calculate_bollinger_bands()
        if upper_band is None:
            return
        
        # Get current position
        user_state = self.info.user_state(self.exchange.account_address)
        current_position = self.get_current_position(user_state)
        
        # Generate signals
        if current_price <= lower_band and current_position <= 0:
            # Price at lower band - potential bounce up
            await self.open_long_position(current_price)
        elif current_price >= upper_band and current_position >= 0:
            # Price at upper band - potential reversion down
            await self.open_short_position(current_price)
        elif current_position != 0 and abs(current_price - middle_band) / middle_band < 0.005:
            # Price back to middle - close position
            await self.close_position()
    
    async def open_long_position(self, price):
        """Open long position"""
        try:
            result = self.exchange.place_order({
                "coin": self.symbol,
                "is_buy": True,
                "sz": self.position_size,
                "limit_px": price * 1.001,  # Small slippage
                "order_type": {"limit": {"tif": "Ioc"}},
                "reduce_only": False
            })
            logging.info(f"Long position opened at ${price:.2f}")
        except Exception as e:
            logging.error(f"Failed to open long position: {str(e)}")
    
    async def open_short_position(self, price):
        """Open short position"""
        try:
            result = self.exchange.place_order({
                "coin": self.symbol,
                "is_buy": False,
                "sz": self.position_size,
                "limit_px": price * 0.999,  # Small slippage
                "order_type": {"limit": {"tif": "Ioc"}},
                "reduce_only": False
            })
            logging.info(f"Short position opened at ${price:.2f}")
        except Exception as e:
            logging.error(f"Failed to open short position: {str(e)}")
```

## Risk Management

### Position Size Calculator

```python
class RiskManager:
    def __init__(self, config):
        self.max_risk_per_trade = config.get('max_risk_per_trade', 0.02)  # 2%
        self.max_portfolio_risk = config.get('max_portfolio_risk', 0.1)   # 10%
        self.max_correlation_exposure = config.get('max_correlation_exposure', 0.3)  # 30%
        
    def calculate_position_size(self, account_value, entry_price, stop_loss_price):
        """Calculate position size based on risk management rules"""
        if stop_loss_price is None or entry_price == stop_loss_price:
            return 0
        
        # Calculate risk per unit
        risk_per_unit = abs(entry_price - stop_loss_price)
        
        # Calculate maximum risk amount
        max_risk_amount = account_value * self.max_risk_per_trade
        
        # Calculate position size
        position_size = max_risk_amount / risk_per_unit
        
        return position_size
    
    def check_risk_limits(self, account_info, new_position_risk):
        """Check if new position violates risk limits"""
        # Calculate current portfolio risk
        current_risk = self.calculate_portfolio_risk(account_info)
        
        # Check portfolio risk limit
        if current_risk + new_position_risk > self.max_portfolio_risk:
            return False, "Portfolio risk limit exceeded"
        
        # Check position concentration
        account_value = float(account_info['marginSummary']['accountValue'])
        if new_position_risk / account_value > self.max_risk_per_trade:
            return False, "Single position risk limit exceeded"
        
        return True, "Risk limits OK"
    
    def calculate_portfolio_risk(self, account_info):
        """Calculate current portfolio risk"""
        total_risk = 0
        account_value = float(account_info['marginSummary']['accountValue'])
        
        for position in account_info.get('assetPositions', []):
            position_value = abs(float(position['position']['positionValue']))
            total_risk += position_value / account_value
        
        return total_risk
```

### Stop Loss and Take Profit Manager

```python
class StopLossManager:
    def __init__(self, config):
        self.exchange = Exchange(config, base_url=config['api_url'])
        self.info = Info(config['api_url'], skip_ws=False)
        self.stop_loss_percentage = config.get('stop_loss_percentage', 0.05)  # 5%
        self.take_profit_percentage = config.get('take_profit_percentage', 0.10)  # 10%
        self.trailing_stop_percentage = config.get('trailing_stop_percentage', 0.03)  # 3%
        self.active_stops = {}
        
    async def set_stop_loss(self, symbol, position_size, entry_price, is_long):
        """Set stop loss order"""
        if is_long:
            stop_price = entry_price * (1 - self.stop_loss_percentage)
            side = False  # Sell to close long
        else:
            stop_price = entry_price * (1 + self.stop_loss_percentage)
            side = True   # Buy to close short
        
        try:
            result = self.exchange.place_order({
                "coin": symbol,
                "is_buy": side,
                "sz": abs(position_size),
                "limit_px": stop_price,
                "order_type": {
                    "trigger": {
                        "trigger_px": stop_price,
                        "is_market": True,
                        "tpsl": "sl"
                    }
                },
                "reduce_only": True
            })
            
            if result.get('status') == 'ok':
                order_id = result['response']['data']['statuses'][0]['resting']['oid']
                self.active_stops[order_id] = {
                    'symbol': symbol,
                    'type': 'stop_loss',
                    'price': stop_price,
                    'is_long': is_long
                }
                logging.info(f"Stop loss set at ${stop_price:.2f} for {symbol}")
        except Exception as e:
            logging.error(f"Failed to set stop loss: {str(e)}")
    
    async def set_take_profit(self, symbol, position_size, entry_price, is_long):
        """Set take profit order"""
        if is_long:
            tp_price = entry_price * (1 + self.take_profit_percentage)
            side = False  # Sell to close long
        else:
            tp_price = entry_price * (1 - self.take_profit_percentage)
            side = True   # Buy to close short
        
        try:
            result = self.exchange.place_order({
                "coin": symbol,
                "is_buy": side,
                "sz": abs(position_size),
                "limit_px": tp_price,
                "order_type": {
                    "trigger": {
                        "trigger_px": tp_price,
                        "is_market": False,
                        "tpsl": "tp"
                    }
                },
                "reduce_only": True
            })
            
            if result.get('status') == 'ok':
                order_id = result['response']['data']['statuses'][0]['resting']['oid']
                self.active_stops[order_id] = {
                    'symbol': symbol,
                    'type': 'take_profit',
                    'price': tp_price,
                    'is_long': is_long
                }
                logging.info(f"Take profit set at ${tp_price:.2f} for {symbol}")
        except Exception as e:
            logging.error(f"Failed to set take profit: {str(e)}")
    
    async def update_trailing_stop(self, symbol, current_price, position):
        """Update trailing stop loss"""
        is_long = float(position['position']['szi']) > 0
        entry_price = float(position['position']['entryPx'])
        
        if is_long:
            # For long positions, trail stops up
            new_stop = current_price * (1 - self.trailing_stop_percentage)
            if new_stop > entry_price * (1 - self.stop_loss_percentage):
                await self.update_stop_loss(symbol, new_stop, is_long)
        else:
            # For short positions, trail stops down
            new_stop = current_price * (1 + self.trailing_stop_percentage)
            if new_stop < entry_price * (1 + self.stop_loss_percentage):
                await self.update_stop_loss(symbol, new_stop, is_long)
```

## Portfolio Management

### Multi-Asset Portfolio Manager

```typescript
class PortfolioManager {
  private infoClient: hl.InfoClient;
  private exchClient: hl.ExchangeClient;
  private targetAllocations: { [symbol: string]: number };
  private rebalanceThreshold: number;
  private maxLeverage: number;

  constructor(privateKey: string, config: any) {
    const httpTransport = new hl.HttpTransport({ isTestnet: config.isTestnet });
    
    this.infoClient = new hl.InfoClient({ transport: httpTransport });
    this.exchClient = new hl.ExchangeClient({ 
      wallet: privateKey, 
      transport: httpTransport,
      isTestnet: config.isTestnet 
    });
    
    this.targetAllocations = config.targetAllocations;
    this.rebalanceThreshold = config.rebalanceThreshold || 0.05;
    this.maxLeverage = config.maxLeverage || 3;
  }

  async rebalancePortfolio() {
    console.log("Starting portfolio rebalancing...");
    
    // Get current portfolio state
    const clearinghouseState = await this.infoClient.clearinghouseState({ 
      user: await this.getAddress() 
    });
    
    const accountValue = parseFloat(clearinghouseState.marginSummary.accountValue);
    const currentAllocations = this.calculateCurrentAllocations(clearinghouseState);
    
    // Calculate required trades
    const trades = this.calculateRebalanceTrades(currentAllocations, accountValue);
    
    // Execute trades
    for (const trade of trades) {
      if (Math.abs(trade.size) > 0.001) { // Minimum trade size
        await this.executeTrade(trade);
      }
    }
    
    console.log("Portfolio rebalancing completed");
  }

  private calculateCurrentAllocations(clearinghouseState: any): { [symbol: string]: number } {
    const allocations: { [symbol: string]: number } = {};
    const accountValue = parseFloat(clearinghouseState.marginSummary.accountValue);
    
    for (const position of clearinghouseState.assetPositions) {
      const symbol = position.position.coin;
      const positionValue = parseFloat(position.position.positionValue);
      allocations[symbol] = positionValue / accountValue;
    }
    
    return allocations;
  }

  private calculateRebalanceTrades(currentAllocations: { [symbol: string]: number }, accountValue: number) {
    const trades: any[] = [];
    
    for (const [symbol, targetAllocation] of Object.entries(this.targetAllocations)) {
      const currentAllocation = currentAllocations[symbol] || 0;
      const allocationDiff = targetAllocation - currentAllocation;
      
      // Check if rebalancing is needed
      if (Math.abs(allocationDiff) > this.rebalanceThreshold) {
        const tradeValue = allocationDiff * accountValue;
        const currentPrice = await this.getCurrentPrice(symbol);
        const tradeSize = tradeValue / currentPrice;
        
        trades.push({
          symbol,
          size: tradeSize,
          side: tradeSize > 0 ? 'buy' : 'sell',
          currentPrice
        });
      }
    }
    
    return trades;
  }

  private async executeTrade(trade: any) {
    try {
      await this.exchClient.order({
        orders: [{
          a: this.getAssetIndex(trade.symbol),
          b: trade.side === 'buy',
          p: (trade.currentPrice * (trade.side === 'buy' ? 1.001 : 0.999)).toFixed(4),
          s: Math.abs(trade.size).toFixed(6),
          r: false,
          t: { limit: { tif: "Ioc" } }
        }],
        grouping: "na"
      });

      console.log(`Executed ${trade.side} ${Math.abs(trade.size).toFixed(6)} ${trade.symbol} at $${trade.currentPrice.toFixed(2)}`);
    } catch (error) {
      console.error(`Failed to execute trade for ${trade.symbol}: ${error}`);
    }
  }

  async monitorRiskMetrics() {
    const clearinghouseState = await this.infoClient.clearinghouseState({ 
      user: await this.getAddress() 
    });
    
    const accountValue = parseFloat(clearinghouseState.marginSummary.accountValue);
    const totalMarginUsed = parseFloat(clearinghouseState.marginSummary.totalMarginUsed);
    const leverage = totalMarginUsed / accountValue;
    
    console.log(`Portfolio Metrics:`);
    console.log(`Account Value: $${accountValue.toFixed(2)}`);
    console.log(`Leverage: ${leverage.toFixed(2)}x`);
    console.log(`Margin Utilization: ${(leverage / this.maxLeverage * 100).toFixed(1)}%`);
    
    // Risk alerts
    if (leverage > this.maxLeverage * 0.8) {
      console.log("⚠️ WARNING: Approaching maximum leverage limit");
    }
    
    if (leverage > this.maxLeverage) {
      console.log("🚨 CRITICAL: Maximum leverage exceeded - reducing positions");
      await this.reducePositions();
    }
  }

  private async reducePositions() {
    // Emergency position reduction logic
    const clearinghouseState = await this.infoClient.clearinghouseState({ 
      user: await this.getAddress() 
    });
    
    // Sort positions by size and reduce largest positions first
    const positions = clearinghouseState.assetPositions
      .filter((p: any) => parseFloat(p.position.szi) !== 0)
      .sort((a: any, b: any) => 
        Math.abs(parseFloat(b.position.positionValue)) - Math.abs(parseFloat(a.position.positionValue))
      );
    
    for (const position of positions.slice(0, 3)) { // Reduce top 3 positions
      const positionSize = parseFloat(position.position.szi);
      const reductionSize = Math.abs(positionSize) * 0.3; // Reduce by 30%
      
      await this.exchClient.order({
        orders: [{
          a: this.getAssetIndex(position.position.coin),
          b: positionSize < 0, // Opposite side to reduce
          p: "0", // Market order
          s: reductionSize.toFixed(6),
          r: true, // Reduce only
          t: { market: {} }
        }],
        grouping: "na"
      });
    }
  }
}
```

This comprehensive documentation provides practical implementations of various trading strategies that can be used with the Hyperliquid platform. Each strategy includes risk management features and can be adapted based on specific trading requirements and market conditions. 