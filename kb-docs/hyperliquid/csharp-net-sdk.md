# HyperLiquid.Net C# SDK Documentation

## Overview

**HyperLiquid.Net** is a comprehensive .NET client library for the HyperLiquid DEX REST and WebSocket APIs. It provides full support for both spot and futures trading with a clean, type-safe C# interface that follows .NET conventions and best practices.

## Installation

### NuGet Package Manager

```bash
dotnet add package HyperLiquid.Net
```

### Package Manager Console

```powershell
Install-Package HyperLiquid.Net
```

## Configuration & Setup

### Dependency Injection Setup

```csharp
using HyperLiquid.Net;

// Method 1: Configuration from code
builder.Services.AddHyperLiquid(options => {
    options.ApiCredentials = new ApiCredentials("YOUR_ADDRESS", "YOUR_PRIVATE_KEY");
});

// Method 2: Configuration from appsettings.json
builder.Services.AddHyperLiquid(builder.Configuration.GetSection("HyperLiquid"));
```

### Direct Client Instantiation

```csharp
using HyperLiquid.Net;

// REST Client
var restClient = new HyperLiquidRestClient(options => {
    options.ApiCredentials = new ApiCredentials("YOUR_ADDRESS", "YOUR_PRIVATE_KEY");
});

// WebSocket Client
var socketClient = new HyperLiquidSocketClient();
```

### Configuration File Example

```json
// appsettings.json
{
  "HyperLiquid": {
    "ApiCredentials": {
      "Key": "YOUR_ADDRESS",
      "Secret": "YOUR_PRIVATE_KEY"
    },
    "Environment": "Live", // or "Testnet"
    "RequestTimeout": "00:00:30"
  }
}
```

## Market Data API

### Getting Exchange Information

```csharp
var hyperLiquidClient = new HyperLiquidRestClient();

// Get all supported spot symbols
var spotResult = await hyperLiquidClient.SpotApi.ExchangeData.GetExchangeInfoAsync();
var spotSymbols = spotResult.Data.Symbols;
Console.WriteLine($"Found {spotSymbols.Count()} spot symbols");

// Get all supported futures symbols
var futuresResult = await hyperLiquidClient.FuturesApi.ExchangeData.GetExchangeInfoAsync();
var futuresSymbols = futuresResult.Data.Universe;
Console.WriteLine($"Found {futuresSymbols.Count()} futures symbols");

// Get exchange info with tickers for spot
var spotTickersResult = await hyperLiquidClient.SpotApi.ExchangeData.GetExchangeInfoAndTickersAsync();
foreach (var ticker in spotTickersResult.Data.Tickers)
{
    Console.WriteLine($"{ticker.Symbol}: ${ticker.MidPrice}");
}

// Get exchange info with tickers for futures
var futuresTickersResult = await hyperLiquidClient.FuturesApi.ExchangeData.GetExchangeInfoAndTickersAsync();
foreach (var ticker in futuresTickersResult.Data.Tickers)
{
    Console.WriteLine($"{ticker.Symbol}: ${ticker.MidPrice}, Funding: {ticker.Funding}");
}
```

### Price Data & Market Information

```csharp
// Get current prices for all assets
var pricesResult = await hyperLiquidClient.SpotApi.ExchangeData.GetPricesAsync();
foreach (var price in pricesResult.Data)
{
    Console.WriteLine($"{price.Key}: ${price.Value}");
}

// Get specific symbol ticker (spot)
var hypeTickerResult = await hyperLiquidClient.SpotApi.ExchangeData.GetExchangeInfoAndTickersAsync();
var hypeTicker = hypeTickerResult.Data.Tickers.Single(x => x.Symbol == "HYPE/USDC");
Console.WriteLine($"HYPE/USDC - Price: ${hypeTicker.MidPrice}, Volume: {hypeTicker.DayBaseVolume}");

// Get specific symbol ticker (futures)
var ethTickerResult = await hyperLiquidClient.FuturesApi.ExchangeData.GetExchangeInfoAndTickersAsync();
var ethTicker = ethTickerResult.Data.Tickers.Single(x => x.Symbol == "ETH");
Console.WriteLine($"ETH - Price: ${ethTicker.MidPrice}, Open Interest: {ethTicker.OpenInterest}");

// Get order book (L2)
var orderBookResult = await hyperLiquidClient.SpotApi.ExchangeData.GetOrderBookAsync("HYPE/USDC");
var orderBook = orderBookResult.Data;
Console.WriteLine($"Best Bid: ${orderBook.Levels[0][0].Price} ({orderBook.Levels[0][0].Size})");
Console.WriteLine($"Best Ask: ${orderBook.Levels[1][0].Price} ({orderBook.Levels[1][0].Size})");

// Get asset information
var assetInfoResult = await hyperLiquidClient.SpotApi.ExchangeData.GetAssetInfoAsync("USDC");
var assetInfo = assetInfoResult.Data;
Console.WriteLine($"Total Supply: {assetInfo.TotalSupply}, Circulating: {assetInfo.CirculatingSupply}");
```

### Historical Data

```csharp
// Get candlestick data
var klinesResult = await hyperLiquidClient.SpotApi.ExchangeData.GetKlinesAsync(
    "HYPE/USDC", 
    KlineInterval.OneHour, 
    DateTime.UtcNow.AddDays(-7), 
    DateTime.UtcNow);

foreach (var kline in klinesResult.Data)
{
    Console.WriteLine($"{kline.OpenTime}: O:{kline.Open} H:{kline.High} L:{kline.Low} C:{kline.Close} V:{kline.Volume}");
}

// Get funding rate history (futures)
var fundingHistoryResult = await hyperLiquidClient.FuturesApi.ExchangeData.GetFundingRateHistoryAsync(
    "ETH",
    DateTime.UtcNow.AddDays(-30),
    DateTime.UtcNow);

foreach (var funding in fundingHistoryResult.Data)
{
    Console.WriteLine($"{funding.Time}: {funding.Coin} - Rate: {funding.FundingRate}, Premium: {funding.Premium}");
}
```

## Account Management

### Account Information

```csharp
var hyperLiquidClient = new HyperLiquidRestClient(options => {
    options.ApiCredentials = new ApiCredentials("YOUR_ADDRESS", "YOUR_PRIVATE_KEY");
});

// Get spot account balances
var balancesResult = await hyperLiquidClient.SpotApi.Account.GetBalancesAsync();
foreach (var balance in balancesResult.Data.Balances)
{
    if (decimal.Parse(balance.Total) > 0)
    {
        Console.WriteLine($"{balance.Coin}: {balance.Total} (Hold: {balance.Hold})");
    }
}

// Get perpetual futures account info
var accountInfoResult = await hyperLiquidClient.FuturesApi.Account.GetAccountInfoAsync();
var accountInfo = accountInfoResult.Data;
Console.WriteLine($"Account Value: ${accountInfo.MarginSummary.AccountValue}");
Console.WriteLine($"Total Margin Used: ${accountInfo.MarginSummary.TotalMarginUsed}");

// Get open positions
foreach (var position in accountInfo.AssetPositions)
{
    if (decimal.Parse(position.Position.Size) != 0)
    {
        Console.WriteLine($"Position: {position.Position.Coin} - Size: {position.Position.Size}, Entry: ${position.Position.EntryPx}");
    }
}
```

### Account Ledger & History

```csharp
// Get account ledger (transaction history)
var ledgerResult = await hyperLiquidClient.SpotApi.Account.GetAccountLedgerAsync();
foreach (var entry in ledgerResult.Data.Take(10)) // Show last 10 entries
{
    Console.WriteLine($"{entry.Time}: {entry.Delta.Type} - {entry.Hash}");
}

// Get user trades
var tradesResult = await hyperLiquidClient.SpotApi.Trading.GetUserTradesAsync();
foreach (var trade in tradesResult.Data.Take(10))
{
    Console.WriteLine($"{trade.Time}: {trade.Side} {trade.Size} {trade.Coin} @ ${trade.Price}");
}

// Get rate limits
var rateLimitsResult = await hyperLiquidClient.SpotApi.Account.GetRateLimitsAsync();
var rateLimits = rateLimitsResult.Data;
Console.WriteLine($"Requests Used: {rateLimits.RequestsUsed}/{rateLimits.RequestsCap}");
Console.WriteLine($"Cumulative Volume: {rateLimits.CumulativeVolume}");
```

## Trading Operations

### Spot Trading

```csharp
var hyperLiquidClient = new HyperLiquidRestClient(options => {
    options.ApiCredentials = new ApiCredentials("YOUR_ADDRESS", "YOUR_PRIVATE_KEY");
});

// Place a spot limit order
var limitOrderResult = await hyperLiquidClient.SpotApi.Trading.PlaceOrderAsync(
    "HYPE/USDC",
    OrderSide.Buy,
    OrderType.Limit,
    quantity: 1m,
    price: 20m);

if (limitOrderResult.Success)
{
    Console.WriteLine($"Limit order placed successfully: {limitOrderResult.Data.Response.Data.Statuses[0].Resting.OrderId}");
}

// Place a spot market order
var currentPrices = await hyperLiquidClient.SpotApi.ExchangeData.GetPricesAsync();
var symbolPrice = currentPrices.Data.Single(x => x.Key == "HYPE/USDC");

var marketOrderResult = await hyperLiquidClient.SpotApi.Trading.PlaceOrderAsync(
    "HYPE/USDC",
    OrderSide.Buy,
    OrderType.Market,
    quantity: 1m,
    price: symbolPrice.Value);

if (marketOrderResult.Success)
{
    Console.WriteLine("Market order placed successfully");
}

// Get open orders
var openOrdersResult = await hyperLiquidClient.SpotApi.Trading.GetOpenOrdersAsync();
foreach (var order in openOrdersResult.Data)
{
    Console.WriteLine($"Order {order.OrderId}: {order.Side} {order.Size} {order.Coin} @ ${order.LimitPrice}");
}

// Get specific order info
if (openOrdersResult.Data.Any())
{
    var firstOrderId = openOrdersResult.Data.First().OrderId;
    var orderInfoResult = await hyperLiquidClient.SpotApi.Trading.GetOrderAsync(firstOrderId);
    Console.WriteLine($"Order Status: {orderInfoResult.Data.Status}");
}

// Cancel an order
if (openOrdersResult.Data.Any())
{
    var orderToCancel = openOrdersResult.Data.First();
    var cancelResult = await hyperLiquidClient.SpotApi.Trading.CancelOrderAsync(
        orderToCancel.Coin, 
        orderToCancel.OrderId);
    
    if (cancelResult.Success)
    {
        Console.WriteLine("Order cancelled successfully");
    }
}
```

### Futures Trading

```csharp
// Place a futures limit order
var futuresOrderResult = await hyperLiquidClient.FuturesApi.Trading.PlaceOrderAsync(
    "ETH",
    OrderSide.Buy,
    OrderType.Limit,
    quantity: 0.1m,
    price: 2000m);

if (futuresOrderResult.Success)
{
    Console.WriteLine("Futures order placed successfully");
}

// Advanced order with reduce-only flag
var reduceOnlyOrderResult = await hyperLiquidClient.FuturesApi.Trading.PlaceOrderAsync(
    symbol: "BTC",
    side: OrderSide.Sell,
    type: OrderType.Limit,
    quantity: 0.01m,
    price: 35000m,
    reduceOnly: true,
    timeInForce: TimeInForce.GoodTillCanceled);

// Place stop-loss order
var stopLossResult = await hyperLiquidClient.FuturesApi.Trading.PlaceOrderAsync(
    symbol: "ETH",
    side: OrderSide.Sell,
    type: OrderType.StopMarket,
    quantity: 0.1m,
    stopPrice: 1800m,
    reduceOnly: true);

// Place take-profit order
var takeProfitResult = await hyperLiquidClient.FuturesApi.Trading.PlaceOrderAsync(
    symbol: "ETH",
    side: OrderSide.Sell,
    type: OrderType.TakeProfitLimit,
    quantity: 0.1m,
    price: 2200m,
    stopPrice: 2200m,
    reduceOnly: true);
```

### Order Management

```csharp
// Get open orders with extended information
var extendedOrdersResult = await hyperLiquidClient.SpotApi.Trading.GetOpenOrdersExtendedAsync();
foreach (var order in extendedOrdersResult.Data)
{
    Console.WriteLine($"Order {order.OrderId}: {order.OrderType} {order.Side} {order.Size} {order.Coin}");
    Console.WriteLine($"  Price: ${order.LimitPrice}, TIF: {order.TimeInForce}");
    Console.WriteLine($"  Reduce Only: {order.ReduceOnly}, Original Size: {order.OriginalSize}");
}

// Cancel all orders for a symbol
var cancelAllResult = await hyperLiquidClient.SpotApi.Trading.CancelAllOrdersAsync("HYPE/USDC");
if (cancelAllResult.Success)
{
    Console.WriteLine("All orders for HYPE/USDC cancelled");
}

// Modify an existing order
if (extendedOrdersResult.Data.Any())
{
    var orderToModify = extendedOrdersResult.Data.First();
    var modifyResult = await hyperLiquidClient.SpotApi.Trading.ModifyOrderAsync(
        orderToModify.OrderId,
        newQuantity: orderToModify.Size * 1.5m, // Increase size by 50%
        newPrice: decimal.Parse(orderToModify.LimitPrice) * 0.95m); // Lower price by 5%
    
    if (modifyResult.Success)
    {
        Console.WriteLine("Order modified successfully");
    }
}
```

## WebSocket Real-time Data

### Market Data Subscriptions

```csharp
var socketClient = new HyperLiquidSocketClient();

// Subscribe to spot ticker updates
var spotSubscription = await socketClient.SpotApi.SubscribeToSymbolUpdatesAsync("HYPE/USDC", (update) =>
{
    Console.WriteLine($"HYPE/USDC Price Update: ${update.Data.MidPrice}");
    Console.WriteLine($"24h Volume: {update.Data.DayBaseVolume}");
});

// Subscribe to futures ticker updates
var futuresSubscription = await socketClient.FuturesApi.SubscribeToSymbolUpdatesAsync("ETH", (update) =>
{
    Console.WriteLine($"ETH Price: ${update.Data.MidPrice}");
    Console.WriteLine($"Funding Rate: {update.Data.Funding}");
    Console.WriteLine($"Open Interest: {update.Data.OpenInterest}");
});

// Subscribe to order book updates
var orderBookSubscription = await socketClient.SpotApi.SubscribeToOrderBookUpdatesAsync("HYPE/USDC", (update) =>
{
    var orderBook = update.Data;
    if (orderBook.Levels[0].Any()) // Bids
    {
        Console.WriteLine($"Best Bid: ${orderBook.Levels[0][0].Price} ({orderBook.Levels[0][0].Size})");
    }
    if (orderBook.Levels[1].Any()) // Asks
    {
        Console.WriteLine($"Best Ask: ${orderBook.Levels[1][0].Price} ({orderBook.Levels[1][0].Size})");
    }
});

// Subscribe to trade updates
var tradesSubscription = await socketClient.SpotApi.SubscribeToTradeUpdatesAsync("HYPE/USDC", (update) =>
{
    foreach (var trade in update.Data)
    {
        Console.WriteLine($"Trade: {trade.Side} {trade.Size} @ ${trade.Price} at {trade.Time}");
    }
});

// Keep the application running
Console.ReadLine();

// Unsubscribe when done
await spotSubscription.CloseAsync();
await futuresSubscription.CloseAsync();
await orderBookSubscription.CloseAsync();
await tradesSubscription.CloseAsync();
```

### Account Data Subscriptions

```csharp
var socketClient = new HyperLiquidSocketClient();

// Subscribe to account updates (requires authentication)
var accountSubscription = await socketClient.SpotApi.SubscribeToAccountUpdatesAsync((update) =>
{
    Console.WriteLine("Account update received:");
    foreach (var balance in update.Data.Balances)
    {
        if (decimal.Parse(balance.Total) > 0)
        {
            Console.WriteLine($"  {balance.Coin}: {balance.Total}");
        }
    }
});

// Subscribe to order updates
var orderSubscription = await socketClient.SpotApi.SubscribeToOrderUpdatesAsync((update) =>
{
    foreach (var order in update.Data)
    {
        Console.WriteLine($"Order Update: {order.Status} - {order.Side} {order.Size} {order.Symbol}");
    }
});

// Subscribe to trade/fill updates
var fillsSubscription = await socketClient.SpotApi.SubscribeToUserTradeUpdatesAsync((update) =>
{
    foreach (var fill in update.Data)
    {
        Console.WriteLine($"Fill: {fill.Side} {fill.Size} {fill.Symbol} @ ${fill.Price}");
        Console.WriteLine($"  Fee: {fill.Fee} {fill.FeeAsset}");
    }
});
```

## Advanced Features

### Batch Operations

```csharp
// Place multiple orders in a single request
var batchOrders = new List<PlaceOrderRequest>
{
    new PlaceOrderRequest
    {
        Symbol = "HYPE/USDC",
        Side = OrderSide.Buy,
        Type = OrderType.Limit,
        Quantity = 1m,
        Price = 19m
    },
    new PlaceOrderRequest
    {
        Symbol = "HYPE/USDC",
        Side = OrderSide.Sell,
        Type = OrderType.Limit,
        Quantity = 1m,
        Price = 21m
    }
};

var batchResult = await hyperLiquidClient.SpotApi.Trading.PlaceBatchOrdersAsync(batchOrders);
if (batchResult.Success)
{
    Console.WriteLine($"Placed {batchOrders.Count} orders successfully");
}

// Cancel multiple orders
var orderIds = new[] { 123456, 123457, 123458 };
var batchCancelResult = await hyperLiquidClient.SpotApi.Trading.CancelBatchOrdersAsync(orderIds);
if (batchCancelResult.Success)
{
    Console.WriteLine($"Cancelled {orderIds.Length} orders");
}
```

### Risk Management

```csharp
public class RiskManager
{
    private readonly HyperLiquidRestClient _client;
    private readonly decimal _maxPositionSize;
    private readonly decimal _maxLoss;

    public RiskManager(HyperLiquidRestClient client, decimal maxPositionSize, decimal maxLoss)
    {
        _client = client;
        _maxPositionSize = maxPositionSize;
        _maxLoss = maxLoss;
    }

    public async Task<bool> CheckRiskLimitsAsync()
    {
        // Check account information
        var accountInfo = await _client.FuturesApi.Account.GetAccountInfoAsync();
        if (!accountInfo.Success)
            return false;

        var marginSummary = accountInfo.Data.MarginSummary;
        var accountValue = decimal.Parse(marginSummary.AccountValue);
        var totalMarginUsed = decimal.Parse(marginSummary.TotalMarginUsed);

        // Check margin utilization
        var marginUtilization = accountValue > 0 ? totalMarginUsed / accountValue : 0;
        if (marginUtilization > 0.8m)
        {
            Console.WriteLine($"⚠️ HIGH RISK: Margin utilization at {marginUtilization:P}");
            return false;
        }

        // Check individual position sizes
        foreach (var position in accountInfo.Data.AssetPositions)
        {
            var positionSize = Math.Abs(decimal.Parse(position.Position.Size));
            if (positionSize > _maxPositionSize)
            {
                Console.WriteLine($"⚠️ Position size limit exceeded for {position.Position.Coin}: {positionSize}");
                return false;
            }
        }

        // Check unrealized PnL
        var totalUnrealizedPnl = accountInfo.Data.AssetPositions
            .Sum(p => decimal.Parse(p.Position.UnrealizedPnl ?? "0"));

        if (totalUnrealizedPnl < -_maxLoss)
        {
            Console.WriteLine($"⚠️ Maximum loss exceeded: ${totalUnrealizedPnl}");
            return false;
        }

        Console.WriteLine("✅ All risk checks passed");
        return true;
    }

    public async Task EmergencyCloseAllPositionsAsync()
    {
        var accountInfo = await _client.FuturesApi.Account.GetAccountInfoAsync();
        if (!accountInfo.Success)
            return;

        foreach (var position in accountInfo.Data.AssetPositions)
        {
            var positionSize = decimal.Parse(position.Position.Size);
            if (positionSize != 0)
            {
                var side = positionSize > 0 ? OrderSide.Sell : OrderSide.Buy;
                var quantity = Math.Abs(positionSize);

                await _client.FuturesApi.Trading.PlaceOrderAsync(
                    position.Position.Coin,
                    side,
                    OrderType.Market,
                    quantity,
                    reduceOnly: true);

                Console.WriteLine($"Emergency close: {side} {quantity} {position.Position.Coin}");
            }
        }
    }
}

// Usage
var riskManager = new RiskManager(hyperLiquidClient, maxPositionSize: 10m, maxLoss: 1000m);
var riskCheckPassed = await riskManager.CheckRiskLimitsAsync();

if (!riskCheckPassed)
{
    await riskManager.EmergencyCloseAllPositionsAsync();
}
```

### Trading Strategy Implementation

```csharp
public class GridTradingStrategy
{
    private readonly HyperLiquidRestClient _client;
    private readonly HyperLiquidSocketClient _socketClient;
    private readonly string _symbol;
    private readonly decimal _gridSize;
    private readonly decimal _orderSize;
    private readonly int _gridLevels;
    private readonly Dictionary<long, GridOrder> _activeOrders = new();

    public GridTradingStrategy(
        HyperLiquidRestClient client,
        HyperLiquidSocketClient socketClient,
        string symbol,
        decimal gridSize,
        decimal orderSize,
        int gridLevels)
    {
        _client = client;
        _socketClient = socketClient;
        _symbol = symbol;
        _gridSize = gridSize;
        _orderSize = orderSize;
        _gridLevels = gridLevels;
    }

    public async Task StartAsync()
    {
        // Get current price
        var tickerResult = await _client.SpotApi.ExchangeData.GetExchangeInfoAndTickersAsync();
        var ticker = tickerResult.Data.Tickers.Single(x => x.Symbol == _symbol);
        var currentPrice = decimal.Parse(ticker.MidPrice);

        // Place initial grid orders
        await PlaceGridOrdersAsync(currentPrice);

        // Subscribe to order updates
        await _socketClient.SpotApi.SubscribeToOrderUpdatesAsync(OnOrderUpdate);

        // Subscribe to price updates for rebalancing
        await _socketClient.SpotApi.SubscribeToSymbolUpdatesAsync(_symbol, OnPriceUpdate);

        Console.WriteLine($"Grid trading strategy started for {_symbol}");
    }

    private async Task PlaceGridOrdersAsync(decimal centerPrice)
    {
        var tasks = new List<Task>();

        // Place buy orders below current price
        for (int i = 1; i <= _gridLevels; i++)
        {
            var price = centerPrice - (_gridSize * i);
            tasks.Add(PlaceGridOrderAsync(OrderSide.Buy, price));
        }

        // Place sell orders above current price
        for (int i = 1; i <= _gridLevels; i++)
        {
            var price = centerPrice + (_gridSize * i);
            tasks.Add(PlaceGridOrderAsync(OrderSide.Sell, price));
        }

        await Task.WhenAll(tasks);
    }

    private async Task PlaceGridOrderAsync(OrderSide side, decimal price)
    {
        try
        {
            var result = await _client.SpotApi.Trading.PlaceOrderAsync(
                _symbol,
                side,
                OrderType.Limit,
                _orderSize,
                price);

            if (result.Success)
            {
                var orderId = result.Data.Response.Data.Statuses[0].Resting.OrderId;
                _activeOrders[orderId] = new GridOrder
                {
                    OrderId = orderId,
                    Side = side,
                    Price = price,
                    Size = _orderSize
                };
                Console.WriteLine($"Placed {side} order at ${price:F2}");
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Failed to place {side} order at ${price:F2}: {ex.Message}");
        }
    }

    private async void OnOrderUpdate(DataEvent<OrderUpdate[]> update)
    {
        foreach (var orderUpdate in update.Data)
        {
            if (_activeOrders.TryGetValue(orderUpdate.OrderId, out var gridOrder))
            {
                if (orderUpdate.Status == OrderStatus.Filled)
                {
                    Console.WriteLine($"Grid order filled: {gridOrder.Side} {gridOrder.Size} @ ${gridOrder.Price:F2}");

                    // Place opposite order
                    var oppositePrice = gridOrder.Side == OrderSide.Buy
                        ? gridOrder.Price + _gridSize
                        : gridOrder.Price - _gridSize;

                    var oppositeSide = gridOrder.Side == OrderSide.Buy ? OrderSide.Sell : OrderSide.Buy;

                    await PlaceGridOrderAsync(oppositeSide, oppositePrice);

                    // Remove filled order from tracking
                    _activeOrders.Remove(orderUpdate.OrderId);
                }
            }
        }
    }

    private void OnPriceUpdate(DataEvent<SymbolTicker> update)
    {
        var currentPrice = decimal.Parse(update.Data.MidPrice);
        Console.WriteLine($"Current price: ${currentPrice:F2}");
        
        // Add rebalancing logic here if needed
    }

    private class GridOrder
    {
        public long OrderId { get; set; }
        public OrderSide Side { get; set; }
        public decimal Price { get; set; }
        public decimal Size { get; set; }
    }
}

// Usage
var gridStrategy = new GridTradingStrategy(
    client: hyperLiquidClient,
    socketClient: socketClient,
    symbol: "HYPE/USDC",
    gridSize: 0.1m,      // $0.10 between grid levels
    orderSize: 1m,       // 1 token per order
    gridLevels: 5);      // 5 levels on each side

await gridStrategy.StartAsync();
```

## Error Handling & Best Practices

### Comprehensive Error Handling

```csharp
public class TradingService
{
    private readonly HyperLiquidRestClient _client;
    private readonly ILogger<TradingService> _logger;

    public TradingService(HyperLiquidRestClient client, ILogger<TradingService> logger)
    {
        _client = client;
        _logger = logger;
    }

    public async Task<bool> PlaceOrderWithRetryAsync(
        string symbol,
        OrderSide side,
        OrderType type,
        decimal quantity,
        decimal? price = null,
        int maxRetries = 3)
    {
        for (int attempt = 1; attempt <= maxRetries; attempt++)
        {
            try
            {
                var result = await _client.SpotApi.Trading.PlaceOrderAsync(
                    symbol, side, type, quantity, price);

                if (result.Success)
                {
                    _logger.LogInformation($"Order placed successfully on attempt {attempt}");
                    return true;
                }
                else
                {
                    _logger.LogWarning($"Order placement failed: {result.Error?.Message}");
                    
                    // Check if it's a retryable error
                    if (!IsRetryableError(result.Error))
                    {
                        _logger.LogError("Non-retryable error, stopping retries");
                        break;
                    }
                }
            }
            catch (HttpRequestException ex) when (ex.Message.Contains("timeout"))
            {
                _logger.LogWarning($"Timeout on attempt {attempt}, retrying...");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, $"Unexpected error on attempt {attempt}");
                
                if (attempt == maxRetries)
                    throw;
            }

            if (attempt < maxRetries)
            {
                var delay = TimeSpan.FromSeconds(Math.Pow(2, attempt)); // Exponential backoff
                await Task.Delay(delay);
            }
        }

        return false;
    }

    private static bool IsRetryableError(Error? error)
    {
        if (error == null) return false;
        
        var retryableErrors = new[]
        {
            "Rate limit exceeded",
            "Internal server error",
            "Service temporarily unavailable",
            "Network timeout"
        };

        return retryableErrors.Any(retryable => 
            error.Message?.Contains(retryable, StringComparison.OrdinalIgnoreCase) == true);
    }
}
```

### Rate Limiting

```csharp
public class RateLimitedClient
{
    private readonly HyperLiquidRestClient _client;
    private readonly SemaphoreSlim _semaphore;
    private readonly Queue<DateTime> _requestTimes = new();
    private readonly int _maxRequestsPerSecond;
    private readonly object _lock = new();

    public RateLimitedClient(HyperLiquidRestClient client, int maxRequestsPerSecond = 10)
    {
        _client = client;
        _maxRequestsPerSecond = maxRequestsPerSecond;
        _semaphore = new SemaphoreSlim(maxRequestsPerSecond, maxRequestsPerSecond);
    }

    public async Task<T> ExecuteWithRateLimitAsync<T>(Func<Task<T>> operation)
    {
        await _semaphore.WaitAsync();
        
        try
        {
            // Check rate limit
            await EnforceRateLimitAsync();
            
            // Execute the operation
            return await operation();
        }
        finally
        {
            _semaphore.Release();
        }
    }

    private async Task EnforceRateLimitAsync()
    {
        lock (_lock)
        {
            var now = DateTime.UtcNow;
            
            // Remove requests older than 1 second
            while (_requestTimes.Count > 0 && (now - _requestTimes.Peek()).TotalSeconds > 1)
            {
                _requestTimes.Dequeue();
            }
            
            // Add current request
            _requestTimes.Enqueue(now);
        }

        // Wait if we've exceeded the rate limit
        if (_requestTimes.Count >= _maxRequestsPerSecond)
        {
            var oldestRequest = _requestTimes.Peek();
            var waitTime = TimeSpan.FromSeconds(1) - (DateTime.UtcNow - oldestRequest);
            
            if (waitTime > TimeSpan.Zero)
            {
                await Task.Delay(waitTime);
            }
        }
    }
}

// Usage
var rateLimitedClient = new RateLimitedClient(hyperLiquidClient, maxRequestsPerSecond: 5);

var result = await rateLimitedClient.ExecuteWithRateLimitAsync(async () =>
    await hyperLiquidClient.SpotApi.ExchangeData.GetExchangeInfoAndTickersAsync());
```

### Configuration Management

```csharp
public class HyperLiquidConfiguration
{
    public string ApiAddress { get; set; } = string.Empty;
    public string PrivateKey { get; set; } = string.Empty;
    public bool UseTestnet { get; set; } = false;
    public TimeSpan RequestTimeout { get; set; } = TimeSpan.FromSeconds(30);
    public int MaxRetries { get; set; } = 3;
    public int RateLimitPerSecond { get; set; } = 10;
}

public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddHyperLiquidServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Bind configuration
        services.Configure<HyperLiquidConfiguration>(
            configuration.GetSection("HyperLiquid"));

        // Register clients
        services.AddSingleton<HyperLiquidRestClient>(provider =>
        {
            var config = provider.GetRequiredService<IOptions<HyperLiquidConfiguration>>().Value;
            return new HyperLiquidRestClient(options =>
            {
                options.ApiCredentials = new ApiCredentials(config.ApiAddress, config.PrivateKey);
                options.RequestTimeout = config.RequestTimeout;
            });
        });

        services.AddSingleton<HyperLiquidSocketClient>();
        services.AddScoped<TradingService>();
        services.AddScoped<RiskManager>();

        return services;
    }
}
```

This comprehensive documentation covers all aspects of the HyperLiquid.Net C# SDK, from basic setup and configuration to advanced trading strategies and error handling. The SDK provides a robust, type-safe interface for .NET developers to interact with the HyperLiquid DEX programmatically. 