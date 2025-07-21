# Hyperliquid TypeScript SDK Documentation

## Overview

This documentation covers multiple TypeScript/JavaScript SDKs for interacting with the Hyperliquid decentralized exchange. We'll focus on two main SDKs:

1. **@nomeida/hyperliquid** - Community SDK with WebSocket support
2. **@nktkas/hyperliquid** - Advanced TypeScript SDK with comprehensive features

## Table of Contents

- [Installation](#installation)
- [Nomeida SDK](#nomeida-sdk)
- [NKTKAS SDK](#nktkas-sdk)
- [WebSocket Integration](#websocket-integration)
- [Trading Examples](#trading-examples)
- [Advanced Features](#advanced-features)
- [Best Practices](#best-practices)

## Installation

### Package Managers

```bash
# npm
npm install hyperliquid
npm install @nktkas/hyperliquid

# yarn
yarn add hyperliquid
yarn add @nktkas/hyperliquid

# pnpm
pnpm add hyperliquid
pnpm add @nktkas/hyperliquid

# bun
bun install hyperliquid
bun install @nktkas/hyperliquid

# Deno
deno add jsr:@nktkas/hyperliquid
```

### Browser Usage

```html
<!-- ESM bundle -->
<script type="module">
  import { Hyperliquid } from "https://unpkg.com/hyperliquid/dist/browser.js";
</script>

<!-- Global bundle (UMD) -->
<script src="https://unpkg.com/hyperliquid/dist/browser.global.js"></script>
<script>
  const sdk = new HyperliquidSDK.Hyperliquid({
    testnet: true,
    enableWs: true
  });
</script>

<!-- ESM from CDN -->
<script type="module">
  import * as hl from "https://esm.sh/jsr/@nktkas/hyperliquid";
</script>
```

## Nomeida SDK

### Basic Setup

```typescript
import { Hyperliquid } from 'hyperliquid';

// Basic initialization
const sdk = new Hyperliquid({
  enableWs: true, // Enable WebSocket functionality
  privateKey: "0x...", // Optional: for trading operations
  testnet: false, // Use mainnet
  walletAddress: "0x...", // Required if using API Agent Wallet
  vaultAddress: "0x...", // Optional: for vault operations
  maxReconnectAttempts: 5 // WebSocket reconnection attempts
});

// Initialize connection (optional, auto-connects when needed)
await sdk.connect();
```

### Market Data Operations

```typescript
// Get all market mid prices
const allMids = await sdk.info.getAllMids();
console.log(allMids);

// Get user's open orders
const openOrders = await sdk.info.getUserOpenOrders('user_address_here');
console.log(openOrders);

// Get L2 order book
const l2Book = await sdk.info.getL2Book('BTC-PERP');
console.log(l2Book);

// Get perpetuals metadata
const perpsMeta = await sdk.info.perpetuals.getMeta();
console.log(perpsMeta);

// Get user's clearinghouse state
const clearinghouseState = await sdk.info.perpetuals.getClearinghouseState('user_address_here');
console.log(clearinghouseState);

// Get spot metadata
const spotMeta = await sdk.info.spot.getSpotMeta();
console.log(spotMeta);

// Get spot clearinghouse state
const spotClearinghouseState = await sdk.info.spot.getSpotClearinghouseState('user_address_here');
console.log(spotClearinghouseState);
```

### Trading Operations

```typescript
// Place a single order
const orderResult = await sdk.exchange.placeOrder({
  coin: 'BTC-PERP',
  is_buy: true,
  sz: 1,
  limit_px: 30000,
  order_type: { limit: { tif: 'Gtc' } },
  reduce_only: false,
  vaultAddress: '0x...' // optional
});
console.log(orderResult);

// Place multiple orders
const multiOrderResult = await sdk.exchange.placeOrder({
  orders: [
    {
      coin: 'BTC-PERP',
      is_buy: true,
      sz: 1,
      limit_px: 30000,
      order_type: { limit: { tif: 'Gtc' } },
      reduce_only: false
    },
    {
      coin: 'ETH-PERP',
      is_buy: false,
      sz: 2,
      limit_px: 2000,
      order_type: { limit: { tif: 'Gtc' } },
      reduce_only: false
    }
  ],
  vaultAddress: '0x...',
  grouping: 'normalTpsl',
  builder: {
    address: '0x...',
    fee: 999,
  }
});
console.log(multiOrderResult);

// Cancel an order
const cancelResult = await sdk.exchange.cancelOrder({
  coin: 'BTC-PERP',
  o: 123456 // order ID
});
console.log(cancelResult);

// Reserve additional request weight (rate limiting)
const reserveResult = await sdk.exchange.reserveRequestWeight(1);
console.log(reserveResult);

// Transfer between perpetual and spot accounts
const transferResult = await sdk.exchange.transferBetweenSpotAndPerp(100, true); // 100 USDC from spot to perp
console.log(transferResult);
```

### Custom Methods

```typescript
// Cancel all orders
const cancelAllResult = await sdk.custom.cancelAllOrders();
console.log(cancelAllResult);

// Cancel all orders for a specific symbol
const cancelAllBTCResult = await sdk.custom.cancelAllOrders('BTC-PERP');
console.log(cancelAllBTCResult);

// Get all tradable assets
const allAssets = sdk.custom.getAllAssets();
console.log(allAssets);
```

### WebSocket Subscriptions

```typescript
// Enable WebSocket
const sdk = new Hyperliquid({ enableWs: true });
await sdk.connect();

// Subscribe to all mid prices
sdk.subscriptions.subscribeToAllMids((data) => {
  console.log('All mids update:', data);
});

// Subscribe to user fills
sdk.subscriptions.subscribeToUserFills("wallet_address_here", (data) => {
  console.log('User fills:', data);
});

// Subscribe to candle data
sdk.subscriptions.subscribeToCandle("BTC-PERP", "1m", (data) => {
  console.log('Candle data:', data);
});

// WebSocket POST requests
const l2BookResponse = await sdk.subscriptions.postRequest('info', {
  type: 'l2Book',
  coin: 'BTC'
});
console.log(l2BookResponse);

// Authorized WebSocket requests (requires privateKey)
if (sdk.isAuthenticated()) {
  const orderPayload = await sdk.exchange.getOrderPayload({
    coin: 'BTC',
    is_buy: true,
    sz: '0.001',
    limit_px: '50000',
    order_type: { limit: { tif: 'Gtc' } },
    reduce_only: false
  });
  
  const orderResponse = await sdk.subscriptions.postRequest('action', orderPayload);
  console.log(orderResponse);
}
```

### Error Handling

```typescript
try {
  await sdk.connect();
} catch (error) {
  if (error.message.includes('WebSocket')) {
    console.error('WebSocket connection failed:', error);
  } else if (error.message.includes('network')) {
    console.error('Network error:', error);
  } else {
    console.error('Unknown error:', error);
  }
}

// Automatic trailing zero handling
await sdk.exchange.placeOrder({
  coin: "BTC-PERP",
  is_buy: true,
  sz: "1.0000",  // Will be automatically converted to "1"
  limit_px: "50000.00",  // Will be automatically converted to "50000"
  reduce_only: false,
  order_type: { limit: { tif: 'Gtc' } }
});
```

## NKTKAS SDK

### Transport Layer

```typescript
import * as hl from "@nktkas/hyperliquid";

// HTTP Transport - suitable for one-time requests or serverless environments
const httpTransport = new hl.HttpTransport({
  isTestnet: false,
  timeout: 10000,
  server: {
    mainnet: {
      api: "https://api.hyperliquid.xyz",
      rpc: "https://api.hyperliquid.xyz/rpc"
    }
  },
  onRequest: (request) => {
    console.log('Sending request:', request.url);
    return request;
  },
  onResponse: (response) => {
    console.log('Received response:', response.status);
    return response;
  }
});

// WebSocket Transport - better network latency
const wsTransport = new hl.WebSocketTransport({
  url: "wss://api.hyperliquid.xyz/ws",
  timeout: 10000,
  keepAlive: {
    interval: 30000,
    timeout: 10000
  },
  reconnect: {
    maxRetries: 3,
    connectionTimeout: 10000,
    connectionDelay: (attempt) => Math.min(1000 * Math.pow(2, attempt), 10000),
    shouldReconnect: (error) => true,
    messageBuffer: 'fifo'
  },
  autoResubscribe: true
});
```

### Info Client

```typescript
const transport = new hl.HttpTransport();
const infoClient = new hl.InfoClient({ transport });

// Market data
const allMids = await infoClient.allMids();
const l2Book = await infoClient.l2Book({ coin: "BTC" });
const meta = await infoClient.meta();
const spotMeta = await infoClient.spotMeta();

// Account information
const clearinghouseState = await infoClient.clearinghouseState({ user: "0x..." });
const openOrders = await infoClient.openOrders({ user: "0x..." });
const userFills = await infoClient.userFills({ user: "0x..." });

// Market analysis
const candleSnapshot = await infoClient.candleSnapshot({
  coin: "BTC",
  interval: "1h",
  startTime: 1640995200000,
  endTime: 1641081600000
});

const fundingHistory = await infoClient.fundingHistory({
  coin: "BTC",
  startTime: 1640995200000,
  endTime: 1641081600000
});

// Advanced features
const marginTable = await infoClient.marginTable({ user: "0x..." });
const portfolio = await infoClient.portfolio({ user: "0x..." });
const userFees = await infoClient.userFees({ user: "0x..." });
```

### Exchange Client

```typescript
// Multiple wallet initialization options
const privateKey = "0x...";

// 1. Using private key directly
const exchClient1 = new hl.ExchangeClient({ 
  wallet: privateKey, 
  transport: httpTransport 
});

// 2. Using Viem
import { privateKeyToAccount } from "viem/accounts";
const viemAccount = privateKeyToAccount("0x...");
const exchClient2 = new hl.ExchangeClient({ 
  wallet: viemAccount, 
  transport: httpTransport 
});

// 3. Using Ethers
import { ethers } from "ethers";
const ethersWallet = new ethers.Wallet("0x...");
const exchClient3 = new hl.ExchangeClient({ 
  wallet: ethersWallet, 
  transport: httpTransport 
});

// 4. Using external wallet (MetaMask)
import { createWalletClient, custom } from "viem";
const [account] = await window.ethereum.request({ method: "eth_requestAccounts" });
const externalWallet = createWalletClient({ 
  account, 
  transport: custom(window.ethereum) 
});
const exchClient4 = new hl.ExchangeClient({ 
  wallet: externalWallet, 
  transport: httpTransport 
});

// Trading operations
const orderResult = await exchClient1.order({
  orders: [{
    a: 0,
    b: true,
    p: "30000",
    s: "0.1",
    r: false,
    t: {
      limit: {
        tif: "Gtc",
      },
    },
  }],
  grouping: "na",
});

// Advanced order operations
const batchModifyResult = await exchClient1.batchModify({
  modifies: [
    {
      oid: 123456,
      order: {
        a: 0,
        b: true,
        p: "31000",
        s: "0.2",
        r: false,
        t: { limit: { tif: "Gtc" } }
      }
    }
  ]
});

// Cancel operations
const cancelResult = await exchClient1.cancel({
  cancels: [{ a: 0, o: 123456 }]
});

// Account management
const approveAgentResult = await exchClient1.approveAgent({
  agentAddress: "0x...",
  agentName: "MyAgent",
});

const withdrawResult = await exchClient1.withdraw3({
  destination: "0x...",
  amount: "100",
});

// Leverage and margin
const leverageResult = await exchClient1.updateLeverage({
  asset: 0,
  isCross: true,
  leverage: 10
});

const marginResult = await exchClient1.updateIsolatedMargin({
  asset: 0,
  isBuy: true,
  ntli: "1000"
});
```

### Subscription Client

```typescript
const wsTransport = new hl.WebSocketTransport();
const subsClient = new hl.SubscriptionClient({ transport: wsTransport });

// Market subscriptions
const allMidsSub = await subsClient.allMids((data) => {
  console.log('All mids:', data);
});

const l2BookSub = await subsClient.l2Book({ coin: "BTC" }, (data) => {
  console.log('L2 book:', data);
});

const candleSub = await subsClient.candle({ coin: "BTC", interval: "1h" }, (data) => {
  console.log('Candle:', data);
});

const tradesSub = await subsClient.trades({ coin: "BTC" }, (data) => {
  console.log('Trades:', data);
});

// Account subscriptions
const userFillsSub = await subsClient.userFills({ user: "0x..." }, (data) => {
  console.log('User fills:', data);
});

const orderUpdatesSub = await subsClient.orderUpdates({ user: "0x..." }, (data) => {
  console.log('Order updates:', data);
});

const notificationSub = await subsClient.notification({ user: "0x..." }, (data) => {
  console.log('Notification:', data);
});

// Unsubscribe
await allMidsSub.unsubscribe();
```

### Multi-Signature Client

```typescript
import { privateKeyToAccount } from "viem/accounts";
import { ethers } from "ethers";

const multiSignAddress = "0x...";
const signers = [
  privateKeyToAccount("0x..."), // Leader (must contain own address)
  new ethers.Wallet("0x..."),
  { // Custom async wallet
    async signTypedData(params) {
      // Custom signing logic
      return "0x..."; // return hex signature
    },
  },
  "0x...", // Private key directly
];

const transport = new hl.HttpTransport();
const multiSignClient = new hl.MultiSignClient({ 
  transport, 
  multiSignAddress, 
  signers 
});

// Same API as ExchangeClient
const orderResult = await multiSignClient.order({
  orders: [{
    a: 0,
    b: true,
    p: "30000",
    s: "0.1",
    r: false,
    t: { limit: { tif: "Gtc" } },
  }],
  grouping: "na",
});

const approveResult = await multiSignClient.approveAgent({
  agentAddress: "0x...",
  agentName: "Agent",
});

const withdrawResult = await multiSignClient.withdraw3({
  destination: "0x...",
  amount: "100",
});
```

## Advanced Features

### Custom Signing

```typescript
import { signUserSignedAction, userSignedActionEip712Types } from "@nktkas/hyperliquid/signing";

const privateKey = "0x...";

// Approve agent with custom signing
const action = {
  type: "approveAgent",
  signatureChainId: "0x66eee",
  hyperliquidChain: "Mainnet",
  agentAddress: "0x...",
  agentName: "Agent",
  nonce: Date.now()
} as const;

const signature = await signUserSignedAction({
  wallet: privateKey,
  action,
  types: userSignedActionEip712Types[action.type]
});

// Send to API
const response = await fetch("https://api.hyperliquid.xyz/exchange", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ action, signature, nonce: action.nonce })
});

// Cancel order with custom signing
import { actionSorter, signL1Action } from "@nktkas/hyperliquid/signing";

const cancelAction = {
  type: "cancel",
  cancels: [{ a: 0, o: 12345 }]
} as const;

const nonce = Date.now();
const cancelSignature = await signL1Action({
  wallet: privateKey,
  action: actionSorter[cancelAction.type](cancelAction),
  nonce
});

const cancelResponse = await fetch("https://api.hyperliquid.xyz/exchange", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ action: cancelAction, signature: cancelSignature, nonce })
});
```

### React Native Support

```javascript
// React Native polyfills (for RN 0.76.3 / Expo v52)
import { Event, EventTarget } from "event-target-shim";

if (!globalThis.EventTarget || !globalThis.Event) {
  globalThis.EventTarget = EventTarget;
  globalThis.Event = Event;
}

if (!globalThis.CustomEvent) {
  globalThis.CustomEvent = function (type, params) {
    params = params || {};
    const event = new Event(type, params);
    event.detail = params.detail || null;
    return event;
  };
}

if (!AbortSignal.timeout) {
  AbortSignal.timeout = function (delay) {
    const controller = new AbortController();
    setTimeout(() => controller.abort(), delay);
    return controller.signal;
  };
}

if (!Promise.withResolvers) {
  Promise.withResolvers = function () {
    let resolve, reject;
    const promise = new Promise((res, rej) => {
      resolve = res;
      reject = rej;
    });
    return { promise, resolve, reject };
  };
}
```

## Complete Trading Bot Example

```typescript
import * as hl from "@nktkas/hyperliquid";

class HyperliquidTradingBot {
  private infoClient: hl.InfoClient;
  private exchClient: hl.ExchangeClient;
  private subsClient: hl.SubscriptionClient;
  private activeOrders = new Map();
  
  constructor(privateKey: string, isTestnet = false) {
    const httpTransport = new hl.HttpTransport({ isTestnet });
    const wsTransport = new hl.WebSocketTransport({ 
      url: isTestnet ? "wss://api.hyperliquid-testnet.xyz/ws" : "wss://api.hyperliquid.xyz/ws" 
    });
    
    this.infoClient = new hl.InfoClient({ transport: httpTransport });
    this.exchClient = new hl.ExchangeClient({ 
      wallet: privateKey, 
      transport: httpTransport,
      isTestnet 
    });
    this.subsClient = new hl.SubscriptionClient({ transport: wsTransport });
  }
  
  async start() {
    // Subscribe to market data
    await this.subsClient.allMids((data) => {
      this.onPriceUpdate(data);
    });
    
    // Subscribe to order updates
    await this.subsClient.orderUpdates({ user: await this.getAddress() }, (data) => {
      this.onOrderUpdate(data);
    });
    
    // Subscribe to fills
    await this.subsClient.userFills({ user: await this.getAddress() }, (data) => {
      this.onFill(data);
    });
    
    console.log("Trading bot started");
  }
  
  private async getAddress(): Promise<string> {
    // Implementation to get wallet address
    return "0x...";
  }
  
  private onPriceUpdate(data: any) {
    // Implement your trading logic
    for (const [coin, price] of Object.entries(data)) {
      this.checkTradingOpportunity(coin as string, price as string);
    }
  }
  
  private async checkTradingOpportunity(coin: string, price: string) {
    const numPrice = parseFloat(price);
    
    // Simple mean reversion strategy
    const history = await this.infoClient.candleSnapshot({
      coin,
      interval: "1h",
      startTime: Date.now() - 24 * 60 * 60 * 1000, // 24 hours ago
      endTime: Date.now()
    });
    
    if (history.length > 0) {
      const avgPrice = history.reduce((sum, candle) => sum + parseFloat(candle.c), 0) / history.length;
      const deviation = (numPrice - avgPrice) / avgPrice;
      
      if (deviation < -0.02) { // Price 2% below average
        await this.placeBuyOrder(coin, numPrice);
      } else if (deviation > 0.02) { // Price 2% above average
        await this.placeSellOrder(coin, numPrice);
      }
    }
  }
  
  private async placeBuyOrder(coin: string, price: number) {
    try {
      const result = await this.exchClient.order({
        orders: [{
          a: this.getAssetIndex(coin),
          b: true,
          p: (price * 0.995).toFixed(4), // 0.5% below market
          s: "0.01",
          r: false,
          t: { limit: { tif: "Gtc" } }
        }],
        grouping: "na"
      });
      
      console.log(`Buy order placed for ${coin}:`, result);
    } catch (error) {
      console.error(`Failed to place buy order for ${coin}:`, error);
    }
  }
  
  private async placeSellOrder(coin: string, price: number) {
    try {
      const result = await this.exchClient.order({
        orders: [{
          a: this.getAssetIndex(coin),
          b: false,
          p: (price * 1.005).toFixed(4), // 0.5% above market
          s: "0.01",
          r: false,
          t: { limit: { tif: "Gtc" } }
        }],
        grouping: "na"
      });
      
      console.log(`Sell order placed for ${coin}:`, result);
    } catch (error) {
      console.error(`Failed to place sell order for ${coin}:`, error);
    }
  }
  
  private getAssetIndex(coin: string): number {
    // Map coin symbols to asset indices
    const assetMap: { [key: string]: number } = {
      "BTC": 0,
      "ETH": 1,
      // Add more mappings as needed
    };
    return assetMap[coin] || 0;
  }
  
  private onOrderUpdate(data: any) {
    console.log("Order update:", data);
    // Handle order updates (filled, cancelled, etc.)
  }
  
  private onFill(data: any) {
    console.log("Fill received:", data);
    // Handle trade fills
  }
  
  async stop() {
    // Cancel all open orders
    const openOrders = await this.infoClient.openOrders({ 
      user: await this.getAddress() 
    });
    
    if (openOrders.length > 0) {
      const cancels = openOrders.map(order => ({
        a: order.coin,
        o: order.oid
      }));
      
      await this.exchClient.cancel({ cancels });
      console.log("All orders cancelled");
    }
  }
}

// Usage
const bot = new HyperliquidTradingBot("0x...", true); // testnet
await bot.start();

// Stop bot after some time
setTimeout(async () => {
  await bot.stop();
}, 60000); // Stop after 1 minute
```

## Best Practices

### Error Handling

```typescript
const maxRetries = 3;
const retryDelay = 1000;

async function withRetry<T>(operation: () => Promise<T>): Promise<T> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (attempt === maxRetries) throw error;
      
      console.warn(`Attempt ${attempt} failed, retrying in ${retryDelay}ms...`);
      await new Promise(resolve => setTimeout(resolve, retryDelay * attempt));
    }
  }
  throw new Error("Max retries exceeded");
}

// Usage
const result = await withRetry(() => 
  sdk.exchange.placeOrder({
    coin: 'BTC-PERP',
    is_buy: true,
    sz: 0.1,
    limit_px: 30000,
    order_type: { limit: { tif: 'Gtc' } },
    reduce_only: false
  })
);
```

### Rate Limiting

```typescript
class RateLimiter {
  private requests: number[] = [];
  
  constructor(private maxRequests: number, private windowMs: number) {}
  
  async checkLimit(): Promise<void> {
    const now = Date.now();
    this.requests = this.requests.filter(time => now - time < this.windowMs);
    
    if (this.requests.length >= this.maxRequests) {
      const oldestRequest = Math.min(...this.requests);
      const waitTime = this.windowMs - (now - oldestRequest);
      await new Promise(resolve => setTimeout(resolve, waitTime));
      return this.checkLimit();
    }
    
    this.requests.push(now);
  }
}

const rateLimiter = new RateLimiter(10, 1000); // 10 requests per second

async function rateLimit<T>(operation: () => Promise<T>): Promise<T> {
  await rateLimiter.checkLimit();
  return operation();
}
```

### Connection Management

```typescript
class ConnectionManager {
  private sdk: any;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  
  constructor(config: any) {
    this.sdk = new Hyperliquid(config);
  }
  
  async connect(): Promise<void> {
    try {
      await this.sdk.connect();
      this.reconnectAttempts = 0;
      console.log("Connected successfully");
    } catch (error) {
      if (this.reconnectAttempts < this.maxReconnectAttempts) {
        this.reconnectAttempts++;
        const delay = Math.pow(2, this.reconnectAttempts) * 1000;
        console.log(`Connection failed, retrying in ${delay}ms...`);
        setTimeout(() => this.connect(), delay);
      } else {
        throw new Error("Max reconnection attempts exceeded");
      }
    }
  }
  
  async ensureConnection(): Promise<void> {
    if (!this.sdk.isConnected()) {
      await this.connect();
    }
  }
}
```

This comprehensive documentation covers both major TypeScript SDKs for Hyperliquid, providing detailed examples for market data retrieval, trading operations, WebSocket integration, and advanced features like multi-signature support and custom signing. The examples demonstrate real-world usage patterns and best practices for building robust trading applications. 