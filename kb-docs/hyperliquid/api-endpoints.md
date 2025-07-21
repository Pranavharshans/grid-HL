# Hyperliquid API Endpoints Documentation

## Overview

Hyperliquid provides both REST and WebSocket APIs for accessing market data, account information, and trading functionality. This documentation covers all available endpoints with detailed descriptions, parameters, and response formats.

## Base URLs

- **Mainnet API**: `https://api.hyperliquid.xyz`
- **Testnet API**: `https://api.hyperliquid-testnet.xyz`
- **WebSocket Mainnet**: `wss://api.hyperliquid.xyz/ws`
- **WebSocket Testnet**: `wss://api.hyperliquid-testnet.xyz/ws`

## Authentication

Hyperliquid uses EIP-712 signature-based authentication for trading operations. Public endpoints (market data) don't require authentication.

### Signature Process

1. Create the action object
2. Sort the action using the appropriate sorter
3. Sign with EIP-712 using your private key
4. Include signature and nonce in the request

```typescript
// Example signing process
import { signL1Action, actionSorter } from "@nktkas/hyperliquid/signing";

const action = {
  type: "order",
  orders: [{ /* order details */ }],
  grouping: "na"
};

const nonce = Date.now();
const signature = await signL1Action({
  wallet: privateKey,
  action: actionSorter[action.type](action),
  nonce
});
```

## REST API Endpoints

### Info Endpoints (Public)

#### POST /info

The main endpoint for retrieving public information. The request body determines the type of data returned.

##### Get All Mids

```http
POST /info
Content-Type: application/json

{
  "type": "allMids"
}
```

**Response:**
```json
{
  "BTC": "43250.0",
  "ETH": "2150.5",
  "SOL": "85.2"
}
```

##### Get L2 Order Book

```http
POST /info
Content-Type: application/json

{
  "type": "l2Book",
  "coin": "BTC"
}
```

**Response:**
```json
{
  "coin": "BTC",
  "time": 1704067200000,
  "levels": [
    [
      {"px": "43249.0", "sz": "0.5", "n": 2},
      {"px": "43248.0", "sz": "1.2", "n": 3}
    ],
    [
      {"px": "43251.0", "sz": "0.8", "n": 1},
      {"px": "43252.0", "sz": "2.1", "n": 4}
    ]
  ]
}
```

##### Get User State

```http
POST /info
Content-Type: application/json

{
  "type": "clearinghouseState",
  "user": "0x..."
}
```

**Response:**
```json
{
  "marginSummary": {
    "accountValue": "10000.50",
    "totalMarginUsed": "2500.25"
  },
  "assetPositions": [
    {
      "position": {
        "coin": "BTC",
        "szi": "0.1",
        "entryPx": "43000.0",
        "positionValue": "4300.0",
        "unrealizedPnl": "25.0"
      },
      "type": "oneWay"
    }
  ],
  "withdrawable": "7475.25"
}
```

##### Get Open Orders

```http
POST /info
Content-Type: application/json

{
  "type": "openOrders",
  "user": "0x..."
}
```

**Response:**
```json
[
  {
    "coin": "BTC",
    "limitPx": "42000.0",
    "oid": 123456789,
    "side": "B",
    "sz": "0.1",
    "timestamp": 1704067200000
  }
]
```

##### Get User Fills

```http
POST /info
Content-Type: application/json

{
  "type": "userFills",
  "user": "0x..."
}
```

**Response:**
```json
[
  {
    "coin": "BTC",
    "px": "43100.0",
    "sz": "0.05",
    "side": "B",
    "time": 1704067200000,
    "startPosition": "0.05",
    "dir": "Open Long",
    "closedPnl": "0.0",
    "hash": "0x...",
    "oid": 123456789,
    "crossed": true,
    "fee": "2.155",
    "tid": 987654321
  }
]
```

##### Get Candlestick Data

```http
POST /info
Content-Type: application/json

{
  "type": "candleSnapshot",
  "req": {
    "coin": "BTC",
    "interval": "1h",
    "startTime": 1704000000000,
    "endTime": 1704067200000
  }
}
```

**Response:**
```json
[
  {
    "t": 1704063600000,
    "T": 1704067199999,
    "s": "BTC",
    "i": "1h",
    "o": "43000.0",
    "c": "43250.0",
    "h": "43300.0",
    "l": "42950.0",
    "v": "125.5",
    "n": 1250
  }
]
```

##### Get Funding Rate History

```http
POST /info
Content-Type: application/json

{
  "type": "fundingHistory",
  "coin": "BTC",
  "startTime": 1704000000000,
  "endTime": 1704067200000
}
```

**Response:**
```json
[
  {
    "coin": "BTC",
    "fundingRate": "0.0001",
    "premium": "0.00015",
    "time": 1704067200000
  }
]
```

##### Get Perpetuals Metadata

```http
POST /info
Content-Type: application/json

{
  "type": "meta"
}
```

**Response:**
```json
{
  "universe": [
    {
      "name": "BTC",
      "szDecimals": 5,
      "maxLeverage": 50
    },
    {
      "name": "ETH",
      "szDecimals": 4,
      "maxLeverage": 50
    }
  ]
}
```

##### Get Spot Metadata

```http
POST /info
Content-Type: application/json

{
  "type": "spotMeta"
}
```

**Response:**
```json
{
  "universe": [
    {
      "tokens": [1, 0],
      "name": "HYPE/USDC",
      "index": 0,
      "isCanonical": true
    }
  ],
  "tokens": [
    {
      "name": "USDC",
      "szDecimals": 8,
      "weiDecimals": 8,
      "index": 0,
      "tokenId": "0x...",
      "isCanonical": true
    },
    {
      "name": "HYPE",
      "szDecimals": 0,
      "weiDecimals": 5,
      "index": 1,
      "tokenId": "0x...",
      "isCanonical": true
    }
  ]
}
```

### Exchange Endpoints (Authenticated)

#### POST /exchange

The main endpoint for all trading operations. Requires authentication.

##### Place Order

```http
POST /exchange
Content-Type: application/json

{
  "action": {
    "type": "order",
    "orders": [
      {
        "a": 0,
        "b": true,
        "p": "43000",
        "s": "0.1",
        "r": false,
        "t": {
          "limit": {
            "tif": "Gtc"
          }
        }
      }
    ],
    "grouping": "na"
  },
  "nonce": 1704067200000,
  "signature": {
    "r": "0x...",
    "s": "0x...",
    "v": 28
  }
}
```

**Order Parameters:**
- `a`: Asset index (0 for BTC, 1 for ETH, etc.)
- `b`: Boolean, true for buy, false for sell
- `p`: Price as string
- `s`: Size as string
- `r`: Reduce only flag
- `t`: Order type object

**Response:**
```json
{
  "status": "ok",
  "response": {
    "type": "order",
    "data": {
      "statuses": [
        {
          "resting": {
            "oid": 123456789
          }
        }
      ]
    }
  }
}
```

##### Cancel Order

```http
POST /exchange
Content-Type: application/json

{
  "action": {
    "type": "cancel",
    "cancels": [
      {
        "a": 0,
        "o": 123456789
      }
    ]
  },
  "nonce": 1704067200000,
  "signature": {
    "r": "0x...",
    "s": "0x...",
    "v": 28
  }
}
```

**Response:**
```json
{
  "status": "ok",
  "response": {
    "type": "cancel",
    "data": {
      "statuses": ["success"]
    }
  }
}
```

##### Modify Order

```http
POST /exchange
Content-Type: application/json

{
  "action": {
    "type": "batchModify",
    "modifies": [
      {
        "oid": 123456789,
        "order": {
          "a": 0,
          "b": true,
          "p": "42500",
          "s": "0.15",
          "r": false,
          "t": {
            "limit": {
              "tif": "Gtc"
            }
          }
        }
      }
    ]
  },
  "nonce": 1704067200000,
  "signature": {
    "r": "0x...",
    "s": "0x...",
    "v": 28
  }
}
```

##### Update Leverage

```http
POST /exchange
Content-Type: application/json

{
  "action": {
    "type": "updateLeverage",
    "asset": 0,
    "isCross": true,
    "leverage": 10
  },
  "nonce": 1704067200000,
  "signature": {
    "r": "0x...",
    "s": "0x...",
    "v": 28
  }
}
```

##### Withdraw USDC

```http
POST /exchange
Content-Type: application/json

{
  "action": {
    "type": "withdraw3",
    "hyperliquidChain": "Mainnet",
    "signatureChainId": "0x66eee",
    "destination": "0x...",
    "amount": "100",
    "time": 1704067200000
  },
  "nonce": 1704067200000,
  "signature": {
    "r": "0x...",
    "s": "0x...",
    "v": 28
  }
}
```

##### Transfer Between Spot and Perp

```http
POST /exchange
Content-Type: application/json

{
  "action": {
    "type": "usdClassTransfer",
    "hyperliquidChain": "Mainnet",
    "signatureChainId": "0x66eee",
    "amount": "100",
    "toPerp": true
  },
  "nonce": 1704067200000,
  "signature": {
    "r": "0x...",
    "s": "0x...",
    "v": 28
  }
}
```

## WebSocket API

### Connection

Connect to the WebSocket endpoint and send/receive JSON messages.

```javascript
const ws = new WebSocket('wss://api.hyperliquid.xyz/ws');

ws.onopen = () => {
  console.log('Connected to Hyperliquid WebSocket');
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Received:', data);
};
```

### Subscriptions

#### Subscribe to All Mids

```json
{
  "method": "subscribe",
  "subscription": {
    "type": "allMids"
  }
}
```

**Response:**
```json
{
  "channel": "allMids",
  "data": {
    "BTC": "43250.0",
    "ETH": "2150.5"
  }
}
```

#### Subscribe to L2 Book

```json
{
  "method": "subscribe",
  "subscription": {
    "type": "l2Book",
    "coin": "BTC"
  }
}
```

**Response:**
```json
{
  "channel": "l2Book",
  "data": {
    "coin": "BTC",
    "time": 1704067200000,
    "levels": [
      [{"px": "43249.0", "sz": "0.5", "n": 2}],
      [{"px": "43251.0", "sz": "0.8", "n": 1}]
    ]
  }
}
```

#### Subscribe to Trades

```json
{
  "method": "subscribe",
  "subscription": {
    "type": "trades",
    "coin": "BTC"
  }
}
```

**Response:**
```json
{
  "channel": "trades",
  "data": [
    {
      "coin": "BTC",
      "side": "B",
      "px": "43250.0",
      "sz": "0.1",
      "time": 1704067200000,
      "hash": "0x..."
    }
  ]
}
```

#### Subscribe to User Events

```json
{
  "method": "subscribe",
  "subscription": {
    "type": "userEvents",
    "user": "0x..."
  }
}
```

**Response:**
```json
{
  "channel": "userEvents",
  "data": {
    "fills": [
      {
        "coin": "BTC",
        "px": "43100.0",
        "sz": "0.05",
        "side": "B",
        "time": 1704067200000,
        "fee": "2.155"
      }
    ]
  }
}
```

#### Subscribe to User Fills

```json
{
  "method": "subscribe",
  "subscription": {
    "type": "userFills",
    "user": "0x..."
  }
}
```

#### Subscribe to Candles

```json
{
  "method": "subscribe",
  "subscription": {
    "type": "candle",
    "coin": "BTC",
    "interval": "1m"
  }
}
```

### WebSocket POST Requests

You can also send HTTP-like requests through WebSocket:

```json
{
  "method": "post",
  "id": 1,
  "request": {
    "type": "info",
    "payload": {
      "type": "l2Book",
      "coin": "BTC"
    }
  }
}
```

## Order Types & Parameters

### Order Types

#### Limit Order
```json
{
  "limit": {
    "tif": "Gtc"  // Good Till Canceled
  }
}
```

#### Market Order
```json
{
  "market": {}
}
```

#### Stop Market Order
```json
{
  "trigger": {
    "triggerPx": "42000",
    "isMarket": true,
    "tpsl": "sl"  // stop-loss
  }
}
```

#### Take Profit Order
```json
{
  "trigger": {
    "triggerPx": "44000",
    "isMarket": false,
    "tpsl": "tp"  // take-profit
  }
}
```

### Time In Force (TIF)

- `Gtc`: Good Till Canceled
- `Ioc`: Immediate Or Cancel
- `Alo`: Add Liquidity Only

### Asset Indices

Common asset indices for perpetuals:
- `0`: BTC
- `1`: ETH
- `2`: SOL
- `3`: MATIC
- `4`: ARB

Use the `/info` endpoint with `type: "meta"` to get the complete list.

## Error Handling

### Common Error Responses

```json
{
  "status": "err",
  "response": "Insufficient margin"
}
```

```json
{
  "status": "err",
  "response": "Invalid signature"
}
```

```json
{
  "status": "err",
  "response": "Order size too small"
}
```

### Rate Limiting

Hyperliquid implements rate limiting to ensure fair usage:

- **Info endpoints**: Higher rate limits for public data
- **Exchange endpoints**: Lower rate limits for trading operations
- **WebSocket**: Connection-based limits

Rate limit headers are included in responses:
- `X-RateLimit-Limit`: Maximum requests per window
- `X-RateLimit-Remaining`: Remaining requests in window
- `X-RateLimit-Reset`: When the window resets

## Best Practices

### 1. Error Handling

Always check the response status and handle errors appropriately:

```javascript
if (response.status === "ok") {
  // Handle success
  console.log("Operation successful:", response.response);
} else {
  // Handle error
  console.error("Operation failed:", response.response);
}
```

### 2. Nonce Management

Use monotonically increasing nonces to prevent replay attacks:

```javascript
let nonce = Date.now();

function getNextNonce() {
  return ++nonce;
}
```

### 3. WebSocket Reconnection

Implement robust reconnection logic for WebSocket connections:

```javascript
class HyperliquidWebSocket {
  constructor(url) {
    this.url = url;
    this.reconnectAttempts = 0;
    this.maxReconnectAttempts = 5;
    this.reconnectDelay = 1000;
  }

  connect() {
    this.ws = new WebSocket(this.url);
    
    this.ws.onopen = () => {
      console.log('Connected');
      this.reconnectAttempts = 0;
      this.resubscribe();
    };

    this.ws.onclose = () => {
      this.handleReconnect();
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };
  }

  handleReconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);
      
      setTimeout(() => {
        console.log(`Reconnecting... (attempt ${this.reconnectAttempts})`);
        this.connect();
      }, delay);
    }
  }
}
```

### 4. Signature Verification

Always verify signatures on the client side before sending:

```javascript
import { verifySignature } from "@nktkas/hyperliquid/signing";

const isValid = await verifySignature({
  action,
  signature,
  nonce,
  address: walletAddress
});

if (!isValid) {
  throw new Error("Invalid signature");
}
```

This comprehensive API documentation covers all major Hyperliquid endpoints and provides practical examples for integrating with the platform. Always refer to the latest API documentation for any updates or changes. 