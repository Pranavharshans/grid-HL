# WebSocket Documentation

## Overview

WebSockets provide full-duplex communication channels over a single TCP connection, enabling real-time data exchange between the client and server. This documentation covers implementing WebSocket connections for real-time price updates, order notifications, and live trading data in the Hyperliquid Telegram bot.

## Table of Contents

1. [WebSocket Basics](#websocket-basics)
2. [Server-Side Implementation](#server-side-implementation)
3. [Client-Side Implementation](#client-side-implementation)
4. [Message Protocol Design](#message-protocol-design)
5. [Authentication & Security](#authentication--security)
6. [Connection Management](#connection-management)
7. [Error Handling & Reconnection](#error-handling--reconnection)
8. [Performance & Scaling](#performance--scaling)
9. [Testing WebSocket Connections](#testing-websocket-connections)
10. [Integration with Hyperliquid](#integration-with-hyperliquid)

## WebSocket Basics

### Why WebSockets for Trading Bots?

- **Real-time Price Updates**: Instant market data for informed trading decisions
- **Order Status Updates**: Immediate notifications when orders are filled/cancelled
- **Low Latency**: Direct connection reduces communication overhead
- **Bidirectional Communication**: Server can push updates without client requests

### WebSocket vs HTTP Polling

```javascript
// HTTP Polling (inefficient)
setInterval(async () => {
  const prices = await fetch('/api/prices');
  updateUI(prices);
}, 1000); // Poll every second

// WebSocket (efficient)
const ws = new WebSocket('ws://localhost:3000');
ws.on('message', (data) => {
  const prices = JSON.parse(data);
  updateUI(prices);
}); // Real-time updates
```

## Server-Side Implementation

### Basic WebSocket Server with ws Library

```javascript
// server/websocket.js
const WebSocket = require('ws');
const jwt = require('jsonwebtoken');
const hyperliquidService = require('./services/hyperliquid');
const logger = require('./utils/logger');

class WebSocketServer {
  constructor(server) {
    this.wss = new WebSocket.Server({ 
      server,
      path: '/ws',
      verifyClient: this.verifyClient.bind(this)
    });
    
    this.clients = new Map();
    this.subscriptions = new Map();
    this.rooms = new Map(); // For grouping clients
    
    this.setupServer();
    this.setupHyperliquidIntegration();
  }

  verifyClient(info) {
    try {
      const url = new URL(info.req.url, 'http://localhost');
      const token = url.searchParams.get('token');
      
      if (!token) {
        logger.warn('WebSocket connection rejected: No token provided');
        return false;
      }
      
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      info.req.user = decoded;
      return true;
      
    } catch (error) {
      logger.warn('WebSocket connection rejected: Invalid token');
      return false;
    }
  }

  setupServer() {
    this.wss.on('connection', (ws, req) => {
      const clientId = this.generateClientId();
      const user = req.user;
      
      const client = {
        id: clientId,
        ws,
        user,
        subscriptions: new Set(),
        lastPing: Date.now(),
        authenticated: true
      };
      
      this.clients.set(clientId, client);
      
      logger.info('WebSocket client connected', { 
        clientId, 
        userId: user.id,
        ip: req.socket.remoteAddress 
      });

      // Send welcome message
      this.sendMessage(clientId, {
        type: 'welcome',
        clientId,
        user: { id: user.id, username: user.username },
        timestamp: Date.now()
      });

      // Set up event handlers
      ws.on('message', (data) => this.handleMessage(clientId, data));
      ws.on('close', () => this.handleDisconnect(clientId));
      ws.on('error', (error) => this.handleError(clientId, error));
      ws.on('pong', () => this.handlePong(clientId));
    });

    // Heartbeat mechanism
    this.heartbeatInterval = setInterval(() => {
      this.performHeartbeat();
    }, 30000); // Every 30 seconds
  }

  handleMessage(clientId, data) {
    try {
      const message = JSON.parse(data);
      const client = this.clients.get(clientId);
      
      if (!client) return;

      logger.debug('WebSocket message received', { 
        clientId, 
        type: message.type,
        userId: client.user.id
      });

      switch (message.type) {
        case 'subscribe':
          this.handleSubscribe(clientId, message);
          break;
        case 'unsubscribe':
          this.handleUnsubscribe(clientId, message);
          break;
        case 'ping':
          this.sendMessage(clientId, { type: 'pong', timestamp: Date.now() });
          break;
        case 'join_room':
          this.handleJoinRoom(clientId, message);
          break;
        case 'leave_room':
          this.handleLeaveRoom(clientId, message);
          break;
        default:
          logger.warn('Unknown message type', { type: message.type, clientId });
      }
      
    } catch (error) {
      logger.error('Error handling WebSocket message', { 
        error: error.message, 
        clientId 
      });
      
      this.sendMessage(clientId, {
        type: 'error',
        message: 'Invalid message format',
        timestamp: Date.now()
      });
    }
  }

  handleSubscribe(clientId, message) {
    const { channel, params = {} } = message;
    const client = this.clients.get(clientId);
    
    if (!client) return;

    // Validate subscription
    if (!this.isValidSubscription(channel, params, client.user)) {
      this.sendMessage(clientId, {
        type: 'error',
        message: 'Invalid subscription or insufficient permissions',
        channel,
        timestamp: Date.now()
      });
      return;
    }

    // Add to client subscriptions
    const subscriptionKey = this.createSubscriptionKey(channel, params);
    client.subscriptions.add(subscriptionKey);

    // Add to global subscriptions map
    if (!this.subscriptions.has(subscriptionKey)) {
      this.subscriptions.set(subscriptionKey, new Set());
    }
    this.subscriptions.get(subscriptionKey).add(clientId);

    // Start data feed if first subscriber
    if (this.subscriptions.get(subscriptionKey).size === 1) {
      this.startDataFeed(channel, params);
    }

    this.sendMessage(clientId, {
      type: 'subscribed',
      channel,
      params,
      timestamp: Date.now()
    });

    logger.info('Client subscribed', { clientId, channel, params });
  }

  handleUnsubscribe(clientId, message) {
    const { channel, params = {} } = message;
    const client = this.clients.get(clientId);
    
    if (!client) return;

    const subscriptionKey = this.createSubscriptionKey(channel, params);
    client.subscriptions.delete(subscriptionKey);

    if (this.subscriptions.has(subscriptionKey)) {
      this.subscriptions.get(subscriptionKey).delete(clientId);
      
      // Stop data feed if no more subscribers
      if (this.subscriptions.get(subscriptionKey).size === 0) {
        this.stopDataFeed(channel, params);
        this.subscriptions.delete(subscriptionKey);
      }
    }

    this.sendMessage(clientId, {
      type: 'unsubscribed',
      channel,
      params,
      timestamp: Date.now()
    });

    logger.info('Client unsubscribed', { clientId, channel, params });
  }

  isValidSubscription(channel, params, user) {
    const validChannels = [
      'prices',
      'orderbook',
      'trades',
      'user_orders',
      'user_fills',
      'account_updates'
    ];

    if (!validChannels.includes(channel)) {
      return false;
    }

    // Check user-specific subscriptions
    if (channel.startsWith('user_') || channel === 'account_updates') {
      return params.userId === user.id || user.role === 'admin';
    }

    return true;
  }

  createSubscriptionKey(channel, params) {
    return `${channel}:${JSON.stringify(params)}`;
  }

  broadcast(channel, params, data) {
    const subscriptionKey = this.createSubscriptionKey(channel, params);
    const subscribers = this.subscriptions.get(subscriptionKey);
    
    if (!subscribers) return;

    const message = {
      type: 'data',
      channel,
      params,
      data,
      timestamp: Date.now()
    };

    subscribers.forEach(clientId => {
      this.sendMessage(clientId, message);
    });
  }

  sendMessage(clientId, message) {
    const client = this.clients.get(clientId);
    
    if (client && client.ws.readyState === WebSocket.OPEN) {
      client.ws.send(JSON.stringify(message));
    }
  }

  performHeartbeat() {
    this.clients.forEach((client, clientId) => {
      if (Date.now() - client.lastPing > 60000) {
        // Client hasn't responded to ping in 60 seconds
        logger.warn('Removing unresponsive client', { clientId });
        client.ws.terminate();
        this.handleDisconnect(clientId);
      } else if (client.ws.readyState === WebSocket.OPEN) {
        client.ws.ping();
      }
    });
  }

  handlePong(clientId) {
    const client = this.clients.get(clientId);
    if (client) {
      client.lastPing = Date.now();
    }
  }

  handleDisconnect(clientId) {
    const client = this.clients.get(clientId);
    
    if (client) {
      // Remove from all subscriptions
      client.subscriptions.forEach(subscriptionKey => {
        if (this.subscriptions.has(subscriptionKey)) {
          this.subscriptions.get(subscriptionKey).delete(clientId);
          
          if (this.subscriptions.get(subscriptionKey).size === 0) {
            const [channel, paramsStr] = subscriptionKey.split(':');
            const params = JSON.parse(paramsStr);
            this.stopDataFeed(channel, params);
            this.subscriptions.delete(subscriptionKey);
          }
        }
      });

      this.clients.delete(clientId);
      
      logger.info('WebSocket client disconnected', { 
        clientId, 
        userId: client.user.id 
      });
    }
  }

  handleError(clientId, error) {
    logger.error('WebSocket client error', { clientId, error: error.message });
  }

  generateClientId() {
    return Math.random().toString(36).substring(2) + Date.now().toString(36);
  }

  close() {
    if (this.heartbeatInterval) {
      clearInterval(this.heartbeatInterval);
    }
    
    this.wss.close();
    logger.info('WebSocket server closed');
  }
}

module.exports = WebSocketServer;
```

### Integration with Express Server

```javascript
// server/app.js
const express = require('express');
const http = require('http');
const WebSocketServer = require('./websocket');

const app = express();
const server = http.createServer(app);

// Initialize WebSocket server
const wsServer = new WebSocketServer(server);

// Express routes
app.get('/health', (req, res) => {
  res.json({ 
    status: 'healthy',
    websocket: {
      clients: wsServer.clients.size,
      subscriptions: wsServer.subscriptions.size
    }
  });
});

server.listen(3000, () => {
  console.log('Server running on port 3000');
});

// Graceful shutdown
process.on('SIGTERM', () => {
  wsServer.close();
  server.close(() => {
    process.exit(0);
  });
});
```

## Client-Side Implementation

### JavaScript WebSocket Client

```javascript
// client/websocket-client.js
class TradingWebSocket {
  constructor(url, token) {
    this.url = url;
    this.token = token;
    this.ws = null;
    this.isConnected = false;
    this.reconnectAttempts = 0;
    this.maxReconnectAttempts = 5;
    this.reconnectInterval = 1000;
    this.subscriptions = new Map();
    this.eventHandlers = new Map();
    this.messageQueue = [];
    
    this.connect();
  }

  connect() {
    try {
      this.ws = new WebSocket(`${this.url}?token=${this.token}`);
      
      this.ws.onopen = this.handleOpen.bind(this);
      this.ws.onmessage = this.handleMessage.bind(this);
      this.ws.onclose = this.handleClose.bind(this);
      this.ws.onerror = this.handleError.bind(this);
      
    } catch (error) {
      console.error('WebSocket connection failed:', error);
      this.scheduleReconnect();
    }
  }

  handleOpen() {
    console.log('WebSocket connected');
    this.isConnected = true;
    this.reconnectAttempts = 0;
    
    // Process queued messages
    while (this.messageQueue.length > 0) {
      const message = this.messageQueue.shift();
      this.send(message);
    }
    
    // Resubscribe to previous subscriptions
    this.subscriptions.forEach((params, channel) => {
      this.subscribe(channel, params);
    });
    
    this.emit('connected');
  }

  handleMessage(event) {
    try {
      const message = JSON.parse(event.data);
      
      switch (message.type) {
        case 'welcome':
          console.log('Welcome message received:', message);
          this.emit('welcome', message);
          break;
          
        case 'data':
          this.handleDataMessage(message);
          break;
          
        case 'subscribed':
          console.log('Subscribed to:', message.channel);
          this.emit('subscribed', message);
          break;
          
        case 'unsubscribed':
          console.log('Unsubscribed from:', message.channel);
          this.emit('unsubscribed', message);
          break;
          
        case 'error':
          console.error('WebSocket error:', message.message);
          this.emit('error', message);
          break;
          
        case 'pong':
          this.emit('pong', message);
          break;
          
        default:
          console.warn('Unknown message type:', message.type);
      }
      
    } catch (error) {
      console.error('Error parsing WebSocket message:', error);
    }
  }

  handleDataMessage(message) {
    const { channel, params, data } = message;
    const subscriptionKey = this.createSubscriptionKey(channel, params);
    
    // Emit specific channel event
    this.emit(`data:${channel}`, data, params);
    
    // Emit generic data event
    this.emit('data', { channel, params, data });
  }

  handleClose(event) {
    console.log('WebSocket disconnected:', event.code, event.reason);
    this.isConnected = false;
    this.emit('disconnected', event);
    
    if (event.code !== 1000) { // Not a normal closure
      this.scheduleReconnect();
    }
  }

  handleError(error) {
    console.error('WebSocket error:', error);
    this.emit('error', error);
  }

  scheduleReconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      const delay = this.reconnectInterval * Math.pow(2, this.reconnectAttempts - 1);
      
      console.log(`Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})`);
      
      setTimeout(() => {
        this.connect();
      }, delay);
    } else {
      console.error('Max reconnection attempts reached');
      this.emit('maxReconnectAttemptsReached');
    }
  }

  send(message) {
    if (this.isConnected && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(message));
    } else {
      // Queue message for when connection is established
      this.messageQueue.push(message);
    }
  }

  subscribe(channel, params = {}) {
    this.subscriptions.set(channel, params);
    this.send({
      type: 'subscribe',
      channel,
      params
    });
  }

  unsubscribe(channel, params = {}) {
    this.subscriptions.delete(channel);
    this.send({
      type: 'unsubscribe',
      channel,
      params
    });
  }

  ping() {
    this.send({ type: 'ping' });
  }

  // Event handling
  on(event, handler) {
    if (!this.eventHandlers.has(event)) {
      this.eventHandlers.set(event, []);
    }
    this.eventHandlers.get(event).push(handler);
  }

  off(event, handler) {
    if (this.eventHandlers.has(event)) {
      const handlers = this.eventHandlers.get(event);
      const index = handlers.indexOf(handler);
      if (index > -1) {
        handlers.splice(index, 1);
      }
    }
  }

  emit(event, ...args) {
    if (this.eventHandlers.has(event)) {
      this.eventHandlers.get(event).forEach(handler => {
        try {
          handler(...args);
        } catch (error) {
          console.error('Error in event handler:', error);
        }
      });
    }
  }

  createSubscriptionKey(channel, params) {
    return `${channel}:${JSON.stringify(params)}`;
  }

  close() {
    this.subscriptions.clear();
    if (this.ws) {
      this.ws.close(1000, 'Client closing');
    }
  }

  // Utility methods for trading bot
  subscribeToPrices(symbols = []) {
    this.subscribe('prices', { symbols });
  }

  subscribeToOrderBook(symbol, depth = 10) {
    this.subscribe('orderbook', { symbol, depth });
  }

  subscribeToUserOrders(userId) {
    this.subscribe('user_orders', { userId });
  }

  subscribeToUserFills(userId) {
    this.subscribe('user_fills', { userId });
  }
}

// Usage example
const wsClient = new TradingWebSocket('ws://localhost:3000/ws', 'your-jwt-token');

wsClient.on('connected', () => {
  console.log('Connected to trading WebSocket');
  
  // Subscribe to real-time price updates
  wsClient.subscribeToPrices(['BTC', 'ETH', 'SOL']);
  
  // Subscribe to user-specific data
  wsClient.subscribeToUserOrders('user123');
  wsClient.subscribeToUserFills('user123');
});

wsClient.on('data:prices', (prices) => {
  console.log('Price update:', prices);
  updatePriceDisplay(prices);
});

wsClient.on('data:user_orders', (orders) => {
  console.log('Order update:', orders);
  updateOrderDisplay(orders);
});
```

## Message Protocol Design

### Standardized Message Format

```javascript
// Base message structure
const BaseMessage = {
  type: string,      // Message type: 'subscribe', 'data', 'error', etc.
  timestamp: number, // Unix timestamp
  id?: string       // Optional message ID for tracking
};

// Subscription message
const SubscriptionMessage = {
  ...BaseMessage,
  type: 'subscribe' | 'unsubscribe',
  channel: string,   // Channel name: 'prices', 'orderbook', etc.
  params: object     // Channel-specific parameters
};

// Data message
const DataMessage = {
  ...BaseMessage,
  type: 'data',
  channel: string,
  params: object,
  data: any         // Channel-specific data
};

// Error message
const ErrorMessage = {
  ...BaseMessage,
  type: 'error',
  code: string,     // Error code
  message: string,  // Human-readable error message
  details?: any     // Additional error details
};
```

### Channel-Specific Protocols

```javascript
// Price channel protocol
const PriceChannelProtocol = {
  subscribe: {
    type: 'subscribe',
    channel: 'prices',
    params: {
      symbols?: string[],    // Optional: specific symbols to watch
      throttle?: number      // Optional: update frequency in ms
    }
  },
  data: {
    type: 'data',
    channel: 'prices',
    params: { /* subscription params */ },
    data: {
      [symbol: string]: {
        price: number,
        change24h: number,
        volume24h: number,
        timestamp: number
      }
    }
  }
};

// Order book channel protocol
const OrderBookChannelProtocol = {
  subscribe: {
    type: 'subscribe',
    channel: 'orderbook',
    params: {
      symbol: string,       // Required: trading symbol
      depth?: number        // Optional: book depth (default: 10)
    }
  },
  data: {
    type: 'data',
    channel: 'orderbook',
    params: { symbol: 'BTC', depth: 10 },
    data: {
      symbol: 'BTC',
      bids: [[price, size], ...],
      asks: [[price, size], ...],
      timestamp: number
    }
  }
};

// User orders channel protocol
const UserOrdersChannelProtocol = {
  subscribe: {
    type: 'subscribe',
    channel: 'user_orders',
    params: {
      userId: string        // Required: user ID
    }
  },
  data: {
    type: 'data',
    channel: 'user_orders',
    params: { userId: 'user123' },
    data: {
      action: 'created' | 'updated' | 'cancelled' | 'filled',
      order: {
        id: string,
        symbol: string,
        side: 'buy' | 'sell',
        amount: number,
        price: number,
        status: string,
        timestamp: number
      }
    }
  }
};
```

## Authentication & Security

### JWT-based Authentication

```javascript
// WebSocket authentication middleware
const authenticateWebSocket = (ws, req) => {
  try {
    const url = new URL(req.url, 'http://localhost');
    const token = url.searchParams.get('token');
    
    if (!token) {
      ws.close(1008, 'Token required');
      return false;
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    return true;
    
  } catch (error) {
    ws.close(1008, 'Invalid token');
    return false;
  }
};

// Rate limiting for WebSocket connections
const rateLimitWS = new Map();

const checkRateLimit = (clientIP) => {
  const now = Date.now();
  const windowMs = 60000; // 1 minute
  const maxConnections = 10;
  
  if (!rateLimitWS.has(clientIP)) {
    rateLimitWS.set(clientIP, []);
  }
  
  const connections = rateLimitWS.get(clientIP);
  
  // Remove old connections
  const validConnections = connections.filter(time => now - time < windowMs);
  
  if (validConnections.length >= maxConnections) {
    return false;
  }
  
  validConnections.push(now);
  rateLimitWS.set(clientIP, validConnections);
  return true;
};
```

### Message Validation

```javascript
// Message validation schemas
const Joi = require('joi');

const subscriptionSchema = Joi.object({
  type: Joi.string().valid('subscribe', 'unsubscribe').required(),
  channel: Joi.string().valid('prices', 'orderbook', 'user_orders', 'user_fills').required(),
  params: Joi.object().default({})
});

const validateMessage = (message) => {
  const { error, value } = subscriptionSchema.validate(message);
  if (error) {
    throw new Error(`Invalid message: ${error.details[0].message}`);
  }
  return value;
};

// Usage in message handler
handleMessage(clientId, data) {
  try {
    const rawMessage = JSON.parse(data);
    const message = validateMessage(rawMessage);
    
    // Process validated message
    this.processMessage(clientId, message);
    
  } catch (error) {
    this.sendError(clientId, 'INVALID_MESSAGE', error.message);
  }
}
```

## Connection Management

### Connection Pooling and Load Balancing

```javascript
// Connection manager for multiple WebSocket servers
class WebSocketConnectionManager {
  constructor() {
    this.servers = [];
    this.currentServerIndex = 0;
    this.healthChecks = new Map();
  }

  addServer(server) {
    this.servers.push(server);
    this.startHealthCheck(server);
  }

  getNextServer() {
    // Round-robin load balancing
    if (this.servers.length === 0) {
      throw new Error('No available servers');
    }
    
    let attempts = 0;
    while (attempts < this.servers.length) {
      const server = this.servers[this.currentServerIndex];
      this.currentServerIndex = (this.currentServerIndex + 1) % this.servers.length;
      
      if (this.isServerHealthy(server)) {
        return server;
      }
      
      attempts++;
    }
    
    throw new Error('No healthy servers available');
  }

  isServerHealthy(server) {
    const health = this.healthChecks.get(server.id);
    return health && health.status === 'healthy';
  }

  startHealthCheck(server) {
    const interval = setInterval(async () => {
      try {
        const response = await fetch(`${server.httpUrl}/health`);
        const health = await response.json();
        
        this.healthChecks.set(server.id, {
          status: 'healthy',
          lastCheck: Date.now(),
          details: health
        });
        
      } catch (error) {
        this.healthChecks.set(server.id, {
          status: 'unhealthy',
          lastCheck: Date.now(),
          error: error.message
        });
      }
    }, 30000); // Check every 30 seconds
    
    server.healthCheckInterval = interval;
  }
}
```

### Graceful Shutdown

```javascript
// Graceful shutdown handling
class WebSocketServer {
  constructor() {
    this.isShuttingDown = false;
    this.shutdownTimeout = 30000; // 30 seconds
  }

  async gracefulShutdown() {
    if (this.isShuttingDown) return;
    
    console.log('Starting graceful shutdown...');
    this.isShuttingDown = true;
    
    // Stop accepting new connections
    this.wss.close();
    
    // Notify all clients about shutdown
    this.clients.forEach(client => {
      this.sendMessage(client.id, {
        type: 'shutdown',
        message: 'Server is shutting down',
        timestamp: Date.now()
      });
    });
    
    // Wait for clients to disconnect or force close after timeout
    const shutdownPromise = new Promise((resolve) => {
      const checkInterval = setInterval(() => {
        if (this.clients.size === 0) {
          clearInterval(checkInterval);
          resolve();
        }
      }, 1000);
    });
    
    const timeoutPromise = new Promise((resolve) => {
      setTimeout(() => {
        // Force close remaining connections
        this.clients.forEach(client => {
          client.ws.terminate();
        });
        resolve();
      }, this.shutdownTimeout);
    });
    
    await Promise.race([shutdownPromise, timeoutPromise]);
    console.log('Graceful shutdown completed');
  }
}

// Process signal handlers
process.on('SIGTERM', async () => {
  await wsServer.gracefulShutdown();
  process.exit(0);
});

process.on('SIGINT', async () => {
  await wsServer.gracefulShutdown();
  process.exit(0);
});
```

## Error Handling & Reconnection

### Client-Side Reconnection Logic

```javascript
class ReconnectingWebSocket {
  constructor(url, protocols, options = {}) {
    this.url = url;
    this.protocols = protocols;
    this.options = {
      maxReconnectAttempts: 10,
      reconnectInterval: 1000,
      maxReconnectInterval: 30000,
      reconnectDecay: 1.5,
      timeoutInterval: 2000,
      ...options
    };
    
    this.reconnectAttempts = 0;
    this.readyState = WebSocket.CONNECTING;
    this.reconnectTimeout = null;
    this.listeners = {};
    
    this.connect();
  }

  connect() {
    this.ws = new WebSocket(this.url, this.protocols);
    
    this.ws.onopen = (event) => {
      this.readyState = WebSocket.OPEN;
      this.reconnectAttempts = 0;
      this.emit('open', event);
    };
    
    this.ws.onclose = (event) => {
      this.readyState = WebSocket.CLOSED;
      this.emit('close', event);
      
      if (!event.wasClean && this.shouldReconnect()) {
        this.scheduleReconnect();
      }
    };
    
    this.ws.onerror = (event) => {
      this.emit('error', event);
    };
    
    this.ws.onmessage = (event) => {
      this.emit('message', event);
    };
  }

  shouldReconnect() {
    return this.reconnectAttempts < this.options.maxReconnectAttempts;
  }

  scheduleReconnect() {
    if (this.reconnectTimeout) {
      clearTimeout(this.reconnectTimeout);
    }
    
    const timeout = Math.min(
      this.options.reconnectInterval * Math.pow(this.options.reconnectDecay, this.reconnectAttempts),
      this.options.maxReconnectInterval
    );
    
    this.reconnectTimeout = setTimeout(() => {
      this.reconnectAttempts++;
      this.emit('reconnect', this.reconnectAttempts);
      this.connect();
    }, timeout);
  }

  send(data) {
    if (this.readyState === WebSocket.OPEN) {
      this.ws.send(data);
    } else {
      throw new Error('WebSocket is not open');
    }
  }

  close(code, reason) {
    if (this.reconnectTimeout) {
      clearTimeout(this.reconnectTimeout);
    }
    this.ws.close(code, reason);
  }

  addEventListener(type, listener) {
    if (!this.listeners[type]) {
      this.listeners[type] = [];
    }
    this.listeners[type].push(listener);
  }

  removeEventListener(type, listener) {
    if (this.listeners[type]) {
      const index = this.listeners[type].indexOf(listener);
      if (index > -1) {
        this.listeners[type].splice(index, 1);
      }
    }
  }

  emit(type, event) {
    if (this.listeners[type]) {
      this.listeners[type].forEach(listener => listener(event));
    }
  }
}
```

### Server-Side Error Recovery

```javascript
// Error recovery mechanisms
class WebSocketErrorHandler {
  constructor(wsServer) {
    this.wsServer = wsServer;
    this.errorCounts = new Map();
    this.maxErrorsPerClient = 10;
    this.errorWindowMs = 60000; // 1 minute
  }

  handleClientError(clientId, error) {
    const now = Date.now();
    
    if (!this.errorCounts.has(clientId)) {
      this.errorCounts.set(clientId, []);
    }
    
    const errors = this.errorCounts.get(clientId);
    
    // Remove old errors outside the window
    const recentErrors = errors.filter(errorTime => now - errorTime < this.errorWindowMs);
    recentErrors.push(now);
    
    this.errorCounts.set(clientId, recentErrors);
    
    // Check if client has too many errors
    if (recentErrors.length > this.maxErrorsPerClient) {
      logger.warn('Client has too many errors, disconnecting', { clientId });
      this.wsServer.disconnectClient(clientId, 1008, 'Too many errors');
      return;
    }
    
    // Send error response to client
    this.wsServer.sendMessage(clientId, {
      type: 'error',
      code: 'CLIENT_ERROR',
      message: error.message,
      timestamp: now
    });
  }

  handleServerError(error) {
    logger.error('WebSocket server error', { error: error.message, stack: error.stack });
    
    // Broadcast server error to all clients if critical
    if (this.isCriticalError(error)) {
      this.wsServer.broadcast('system', {}, {
        type: 'server_error',
        message: 'Server experiencing issues, please reconnect',
        timestamp: Date.now()
      });
    }
  }

  isCriticalError(error) {
    const criticalErrors = [
      'ECONNREFUSED',
      'ETIMEDOUT',
      'DATABASE_CONNECTION_LOST'
    ];
    
    return criticalErrors.some(code => error.code === code || error.message.includes(code));
  }
}
```

## Performance & Scaling

### Message Compression

```javascript
// WebSocket compression with per-message-deflate
const WebSocket = require('ws');

const wss = new WebSocket.Server({
  port: 8080,
  perMessageDeflate: {
    zlibDeflateOptions: {
      level: 6,
      concurrencyLevel: 10,
      windowBits: 13,
      memLevel: 7
    }
  }
});

// Custom compression for large messages
const zlib = require('zlib');

const compressMessage = (data) => {
  const jsonString = JSON.stringify(data);
  
  if (jsonString.length > 1024) { // Compress messages > 1KB
    const compressed = zlib.deflateSync(jsonString);
    return {
      compressed: true,
      data: compressed.toString('base64')
    };
  }
  
  return {
    compressed: false,
    data: jsonString
  };
};

const decompressMessage = (message) => {
  if (message.compressed) {
    const compressed = Buffer.from(message.data, 'base64');
    const decompressed = zlib.inflateSync(compressed);
    return JSON.parse(decompressed.toString());
  }
  
  return JSON.parse(message.data);
};
```

### Message Throttling

```javascript
// Message throttling to prevent spam
class MessageThrottler {
  constructor() {
    this.clientQueues = new Map();
    this.flushInterval = 100; // Flush every 100ms
    
    setInterval(() => {
      this.flushQueues();
    }, this.flushInterval);
  }

  addMessage(clientId, channel, data) {
    if (!this.clientQueues.has(clientId)) {
      this.clientQueues.set(clientId, new Map());
    }
    
    const clientQueue = this.clientQueues.get(clientId);
    
    // For price updates, keep only the latest
    if (channel === 'prices') {
      clientQueue.set(channel, data);
    } else {
      // For other channels, accumulate messages
      if (!clientQueue.has(channel)) {
        clientQueue.set(channel, []);
      }
      clientQueue.get(channel).push(data);
    }
  }

  flushQueues() {
    this.clientQueues.forEach((queue, clientId) => {
      queue.forEach((data, channel) => {
        if (channel === 'prices') {
          // Send latest price data
          this.sendToClient(clientId, channel, data);
        } else {
          // Send accumulated data
          if (data.length > 0) {
            this.sendToClient(clientId, channel, data);
          }
        }
      });
      
      queue.clear();
    });
  }

  sendToClient(clientId, channel, data) {
    const client = this.wsServer.clients.get(clientId);
    if (client && client.ws.readyState === WebSocket.OPEN) {
      client.ws.send(JSON.stringify({
        type: 'data',
        channel,
        data,
        timestamp: Date.now()
      }));
    }
  }
}
```

## Testing WebSocket Connections

### Unit Testing WebSocket Server

```javascript
// tests/websocket.test.js
const WebSocket = require('ws');
const WebSocketServer = require('../src/websocket');
const jwt = require('jsonwebtoken');

describe('WebSocket Server', () => {
  let server;
  let wsServer;
  
  beforeEach(() => {
    server = require('http').createServer();
    wsServer = new WebSocketServer(server);
    server.listen(0); // Random port
  });

  afterEach(() => {
    wsServer.close();
    server.close();
  });

  it('should accept valid connections', (done) => {
    const token = jwt.sign({ id: 'test-user' }, process.env.JWT_SECRET);
    const port = server.address().port;
    const ws = new WebSocket(`ws://localhost:${port}/ws?token=${token}`);
    
    ws.on('open', () => {
      expect(wsServer.clients.size).toBe(1);
      ws.close();
      done();
    });
  });

  it('should reject connections without token', (done) => {
    const port = server.address().port;
    const ws = new WebSocket(`ws://localhost:${port}/ws`);
    
    ws.on('close', (code) => {
      expect(code).toBe(1008); // Policy violation
      expect(wsServer.clients.size).toBe(0);
      done();
    });
  });

  it('should handle subscription messages', (done) => {
    const token = jwt.sign({ id: 'test-user' }, process.env.JWT_SECRET);
    const port = server.address().port;
    const ws = new WebSocket(`ws://localhost:${port}/ws?token=${token}`);
    
    ws.on('open', () => {
      ws.send(JSON.stringify({
        type: 'subscribe',
        channel: 'prices',
        params: { symbols: ['BTC'] }
      }));
    });
    
    ws.on('message', (data) => {
      const message = JSON.parse(data);
      
      if (message.type === 'subscribed') {
        expect(message.channel).toBe('prices');
        ws.close();
        done();
      }
    });
  });
});
```

### Integration Testing

```javascript
// tests/integration/websocket-integration.test.js
const WebSocketClient = require('../src/client/websocket-client');
const { startTestServer, stopTestServer } = require('../helpers/test-server');

describe('WebSocket Integration', () => {
  let testServer;
  let client;
  
  beforeAll(async () => {
    testServer = await startTestServer();
  });
  
  afterAll(async () => {
    await stopTestServer(testServer);
  });
  
  beforeEach(() => {
    const token = jwt.sign({ id: 'test-user' }, process.env.JWT_SECRET);
    client = new WebSocketClient(testServer.wsUrl, token);
  });
  
  afterEach(() => {
    if (client) {
      client.close();
    }
  });

  it('should receive real-time price updates', (done) => {
    client.on('connected', () => {
      client.subscribeToPrices(['BTC']);
    });
    
    client.on('data:prices', (data) => {
      expect(data).toHaveProperty('BTC');
      expect(data.BTC).toHaveProperty('price');
      done();
    });
    
    // Simulate price update from server
    setTimeout(() => {
      testServer.broadcastPriceUpdate({ BTC: { price: 50000 } });
    }, 100);
  });
});
```

## Integration with Hyperliquid

### Hyperliquid WebSocket Subscription Manager

```javascript
// services/hyperliquid-websocket.js
class HyperliquidWebSocketManager {
  constructor(wsServer) {
    this.wsServer = wsServer;
    this.hyperliquidWS = null;
    this.subscriptions = new Map();
    this.reconnectAttempts = 0;
    this.maxReconnectAttempts = 5;
    
    this.connect();
  }

  async connect() {
    try {
      this.hyperliquidWS = new Hyperliquid({
        enableWs: true,
        privateKey: process.env.HYPERLIQUID_PRIVATE_KEY,
        testnet: process.env.NODE_ENV !== 'production'
      });
      
      await this.hyperliquidWS.connect();
      this.setupSubscriptions();
      this.reconnectAttempts = 0;
      
      logger.info('Connected to Hyperliquid WebSocket');
      
    } catch (error) {
      logger.error('Failed to connect to Hyperliquid:', error);
      this.scheduleReconnect();
    }
  }

  setupSubscriptions() {
    // Subscribe to all mid prices
    this.hyperliquidWS.subscriptions.subscribeToAllMids((data) => {
      this.wsServer.broadcast('prices', {}, this.formatPriceData(data));
    });

    // Subscribe to user fills for authenticated users
    this.subscriptions.forEach((params, subscriptionKey) => {
      if (subscriptionKey.startsWith('user_fills:')) {
        const userAddress = params.userAddress;
        this.hyperliquidWS.subscriptions.subscribeToUserFills(userAddress, (fills) => {
          this.wsServer.broadcast('user_fills', { userAddress }, fills);
        });
      }
    });
  }

  formatPriceData(hlData) {
    const formatted = {};
    
    Object.entries(hlData).forEach(([symbol, price]) => {
      formatted[symbol] = {
        price: parseFloat(price),
        timestamp: Date.now()
      };
    });
    
    return formatted;
  }

  subscribeToUserData(userAddress) {
    const subscriptionKey = `user_fills:${userAddress}`;
    
    if (!this.subscriptions.has(subscriptionKey)) {
      this.subscriptions.set(subscriptionKey, { userAddress });
      
      if (this.hyperliquidWS) {
        this.hyperliquidWS.subscriptions.subscribeToUserFills(userAddress, (fills) => {
          this.wsServer.broadcast('user_fills', { userAddress }, fills);
        });
      }
    }
  }

  unsubscribeFromUserData(userAddress) {
    const subscriptionKey = `user_fills:${userAddress}`;
    this.subscriptions.delete(subscriptionKey);
    
    // Note: Hyperliquid SDK doesn't have unsubscribe methods
    // Would need to track subscriptions and reconnect to remove them
  }

  scheduleReconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
      
      setTimeout(() => {
        this.connect();
      }, delay);
    }
  }

  disconnect() {
    if (this.hyperliquidWS) {
      this.hyperliquidWS.disconnect();
    }
  }
}

// Integration with main WebSocket server
wsServer.setupHyperliquidIntegration = function() {
  this.hlManager = new HyperliquidWebSocketManager(this);
  
  // Handle Hyperliquid-specific subscriptions
  this.startDataFeed = (channel, params) => {
    if (channel === 'user_fills' && params.userAddress) {
      this.hlManager.subscribeToUserData(params.userAddress);
    }
  };
  
  this.stopDataFeed = (channel, params) => {
    if (channel === 'user_fills' && params.userAddress) {
      this.hlManager.unsubscribeFromUserData(params.userAddress);
    }
  };
};
```

## Best Practices Summary

1. **Authentication**: Always validate WebSocket connections with proper tokens
2. **Rate Limiting**: Implement connection and message rate limits
3. **Message Validation**: Validate all incoming messages against schemas
4. **Error Handling**: Implement comprehensive error handling and recovery
5. **Reconnection Logic**: Use exponential backoff for reconnection attempts
6. **Resource Management**: Clean up subscriptions and connections properly
7. **Performance**: Use message throttling and compression for large data
8. **Security**: Validate user permissions for data access
9. **Monitoring**: Log connection events and performance metrics
10. **Testing**: Write comprehensive tests for WebSocket functionality

## Resources

- [WebSocket API Documentation](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [ws Library Documentation](https://github.com/websockets/ws)
- [Socket.IO Documentation](https://socket.io/docs/)
- [WebSocket Security Guide](https://portswigger.net/web-security/websockets)
- [Real-time Web Technologies Guide](https://ably.com/blog/websockets-vs-http-polling-vs-sse) 