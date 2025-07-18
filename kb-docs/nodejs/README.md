# Node.js Documentation

## Overview

Node.js is a JavaScript runtime built on Chrome's V8 JavaScript engine that allows you to build scalable server-side applications. This documentation covers essential Node.js concepts, patterns, and best practices for building a robust Telegram bot backend that interacts with Hyperliquid.

## Table of Contents

1. [Environment Setup](#environment-setup)
2. [Project Structure](#project-structure)
3. [Core Modules](#core-modules)
4. [HTTP Server & API](#http-server--api)
5. [WebSocket Implementation](#websocket-implementation)
6. [Error Handling](#error-handling)
7. [Security](#security)
8. [Performance & Optimization](#performance--optimization)
9. [Testing](#testing)
10. [Deployment](#deployment)

## Environment Setup

### Installation & Version Management

```bash
# Install Node.js via Node Version Manager (recommended)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install --lts
nvm use --lts

# Verify installation
node --version
npm --version
```

### Project Initialization

```bash
mkdir hyperliquid-telegram-bot
cd hyperliquid-telegram-bot

# Initialize package.json
npm init -y

# Install essential dependencies
npm install express cors helmet dotenv
npm install hyperliquid grammy
npm install mongoose pg

# Development dependencies
npm install --save-dev nodemon typescript @types/node
npm install --save-dev jest supertest eslint prettier
```

### Environment Configuration

```javascript
// .env file
NODE_ENV=development
PORT=3000
BOT_TOKEN=your_telegram_bot_token
HYPERLIQUID_PRIVATE_KEY=your_private_key
MONGODB_URI=mongodb://localhost:27017/trading-bot
DATABASE_URL=postgresql://user:password@localhost:5432/trading_db
JWT_SECRET=your_jwt_secret
LOG_LEVEL=info
```

```javascript
// config/index.js
require('dotenv').config();

const config = {
  env: process.env.NODE_ENV || 'development',
  port: parseInt(process.env.PORT, 10) || 3000,
  bot: {
    token: process.env.BOT_TOKEN,
    webhook: {
      url: process.env.WEBHOOK_URL,
      port: parseInt(process.env.WEBHOOK_PORT, 10) || 8443
    }
  },
  hyperliquid: {
    privateKey: process.env.HYPERLIQUID_PRIVATE_KEY,
    testnet: process.env.NODE_ENV !== 'production'
  },
  database: {
    mongodb: process.env.MONGODB_URI,
    postgres: process.env.DATABASE_URL
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: '24h'
  },
  logging: {
    level: process.env.LOG_LEVEL || 'info'
  }
};

// Validate required configuration
const required = ['BOT_TOKEN', 'HYPERLIQUID_PRIVATE_KEY'];
required.forEach(key => {
  if (!process.env[key]) {
    throw new Error(`Missing required environment variable: ${key}`);
  }
});

module.exports = config;
```

## Project Structure

```
hyperliquid-telegram-bot/
├── src/
│   ├── bot/
│   │   ├── handlers/
│   │   │   ├── commands.js
│   │   │   ├── callbacks.js
│   │   │   └── messages.js
│   │   ├── middleware/
│   │   │   ├── auth.js
│   │   │   ├── rateLimit.js
│   │   │   └── validation.js
│   │   └── index.js
│   ├── services/
│   │   ├── hyperliquid.js
│   │   ├── database.js
│   │   └── notifications.js
│   ├── models/
│   │   ├── User.js
│   │   ├── Trade.js
│   │   └── Alert.js
│   ├── routes/
│   │   ├── api.js
│   │   ├── webhook.js
│   │   └── health.js
│   ├── utils/
│   │   ├── logger.js
│   │   ├── validation.js
│   │   └── encryption.js
│   ├── middleware/
│   │   ├── auth.js
│   │   ├── cors.js
│   │   └── errorHandler.js
│   └── app.js
├── tests/
├── config/
├── docs/
├── scripts/
└── package.json
```

## Core Modules

### Logger Utility

```javascript
// src/utils/logger.js
const winston = require('winston');
const config = require('../config');

const logFormat = winston.format.combine(
  winston.format.timestamp(),
  winston.format.errors({ stack: true }),
  winston.format.json()
);

const logger = winston.createLogger({
  level: config.logging.level,
  format: logFormat,
  defaultMeta: { service: 'telegram-bot' },
  transports: [
    new winston.transports.File({ 
      filename: 'logs/error.log', 
      level: 'error' 
    }),
    new winston.transports.File({ 
      filename: 'logs/combined.log' 
    })
  ]
});

if (config.env !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.combine(
      winston.format.colorize(),
      winston.format.simple()
    )
  }));
}

module.exports = logger;
```

### Database Service

```javascript
// src/services/database.js
const mongoose = require('mongoose');
const { Pool } = require('pg');
const config = require('../config');
const logger = require('../utils/logger');

class DatabaseService {
  constructor() {
    this.mongodb = null;
    this.postgres = null;
  }

  async connectMongoDB() {
    try {
      this.mongodb = await mongoose.connect(config.database.mongodb, {
        maxPoolSize: 10,
        serverSelectionTimeoutMS: 5000,
      });
      
      logger.info('Connected to MongoDB');
      
      // Handle connection events
      mongoose.connection.on('error', (err) => {
        logger.error('MongoDB connection error:', err);
      });
      
      mongoose.connection.on('disconnected', () => {
        logger.warn('MongoDB disconnected');
      });
      
    } catch (error) {
      logger.error('Failed to connect to MongoDB:', error);
      throw error;
    }
  }

  async connectPostgreSQL() {
    try {
      this.postgres = new Pool({
        connectionString: config.database.postgres,
        max: 20,
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 2000,
      });

      // Test connection
      const client = await this.postgres.connect();
      await client.query('SELECT NOW()');
      client.release();
      
      logger.info('Connected to PostgreSQL');

      // Handle pool events
      this.postgres.on('error', (err) => {
        logger.error('PostgreSQL pool error:', err);
      });

    } catch (error) {
      logger.error('Failed to connect to PostgreSQL:', error);
      throw error;
    }
  }

  async initialize() {
    await Promise.all([
      this.connectMongoDB(),
      this.connectPostgreSQL()
    ]);
  }

  async close() {
    if (this.mongodb) {
      await mongoose.connection.close();
    }
    if (this.postgres) {
      await this.postgres.end();
    }
    logger.info('Database connections closed');
  }

  // Query helper for PostgreSQL
  async query(text, params) {
    const start = Date.now();
    const res = await this.postgres.query(text, params);
    const duration = Date.now() - start;
    
    logger.debug('Query executed', { 
      text: text.substring(0, 100), 
      duration, 
      rows: res.rowCount 
    });
    
    return res;
  }
}

module.exports = new DatabaseService();
```

### Hyperliquid Service

```javascript
// src/services/hyperliquid.js
const { Hyperliquid } = require('hyperliquid');
const config = require('../config');
const logger = require('../utils/logger');

class HyperliquidService {
  constructor() {
    this.sdk = new Hyperliquid({
      enableWs: true,
      privateKey: config.hyperliquid.privateKey,
      testnet: config.hyperliquid.testnet,
      maxReconnectAttempts: 5
    });
    
    this.connected = false;
  }

  async initialize() {
    try {
      await this.sdk.connect();
      this.connected = true;
      logger.info('Connected to Hyperliquid');
      
      // Set up reconnection handling
      this.sdk.subscriptions.on('disconnect', () => {
        this.connected = false;
        logger.warn('Hyperliquid WebSocket disconnected');
      });
      
      this.sdk.subscriptions.on('reconnect', () => {
        this.connected = true;
        logger.info('Hyperliquid WebSocket reconnected');
      });
      
    } catch (error) {
      logger.error('Failed to connect to Hyperliquid:', error);
      throw error;
    }
  }

  async getAccountState(userAddress) {
    try {
      return await this.sdk.info.perpetuals.getClearinghouseState(userAddress);
    } catch (error) {
      logger.error('Failed to get account state:', error);
      throw error;
    }
  }

  async getAllMids() {
    try {
      return await this.sdk.info.getAllMids();
    } catch (error) {
      logger.error('Failed to get all mids:', error);
      throw error;
    }
  }

  async placeOrder(orderParams) {
    try {
      const result = await this.sdk.exchange.placeOrder(orderParams);
      logger.info('Order placed:', { orderId: result.response?.data?.statuses?.[0] });
      return result;
    } catch (error) {
      logger.error('Failed to place order:', error);
      throw error;
    }
  }

  async cancelOrder(cancelParams) {
    try {
      const result = await this.sdk.exchange.cancelOrder(cancelParams);
      logger.info('Order cancelled:', cancelParams);
      return result;
    } catch (error) {
      logger.error('Failed to cancel order:', error);
      throw error;
    }
  }

  subscribeToUserFills(userAddress, callback) {
    if (!this.connected) {
      throw new Error('Not connected to Hyperliquid');
    }
    
    this.sdk.subscriptions.subscribeToUserFills(userAddress, (data) => {
      logger.debug('User fills received:', { user: userAddress, fills: data.length });
      callback(data);
    });
  }

  subscribeToAllMids(callback) {
    if (!this.connected) {
      throw new Error('Not connected to Hyperliquid');
    }
    
    this.sdk.subscriptions.subscribeToAllMids((data) => {
      logger.debug('All mids received:', { count: Object.keys(data).length });
      callback(data);
    });
  }

  async disconnect() {
    if (this.connected) {
      this.sdk.disconnect();
      this.connected = false;
      logger.info('Disconnected from Hyperliquid');
    }
  }
}

module.exports = new HyperliquidService();
```

## HTTP Server & API

### Express Application Setup

```javascript
// src/app.js
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');

const config = require('./config');
const logger = require('./utils/logger');
const errorHandler = require('./middleware/errorHandler');
const authMiddleware = require('./middleware/auth');

// Route imports
const apiRoutes = require('./routes/api');
const webhookRoutes = require('./routes/webhook');
const healthRoutes = require('./routes/health');

const app = express();

// Security middleware
app.use(helmet());
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || '*',
  credentials: true
}));

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: 'Too many requests from this IP',
  standardHeaders: true,
  legacyHeaders: false,
});
app.use('/api', limiter);

// Body parsing middleware
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// Request logging
app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = Date.now() - start;
    logger.info('HTTP Request', {
      method: req.method,
      url: req.url,
      status: res.statusCode,
      duration,
      ip: req.ip
    });
  });
  next();
});

// Routes
app.use('/health', healthRoutes);
app.use('/webhook', webhookRoutes);
app.use('/api', authMiddleware, apiRoutes);

// 404 handler
app.use('*', (req, res) => {
  res.status(404).json({ error: 'Route not found' });
});

// Error handling middleware
app.use(errorHandler);

module.exports = app;
```

### API Routes

```javascript
// src/routes/api.js
const express = require('express');
const router = express.Router();
const hyperliquidService = require('../services/hyperliquid');
const logger = require('../utils/logger');

// Get user's trading account state
router.get('/account/:address', async (req, res, next) => {
  try {
    const { address } = req.params;
    
    if (!address || !/^0x[a-fA-F0-9]{40}$/.test(address)) {
      return res.status(400).json({ error: 'Invalid wallet address' });
    }

    const accountState = await hyperliquidService.getAccountState(address);
    res.json({ success: true, data: accountState });
    
  } catch (error) {
    next(error);
  }
});

// Get current market prices
router.get('/prices', async (req, res, next) => {
  try {
    const prices = await hyperliquidService.getAllMids();
    res.json({ success: true, data: prices });
  } catch (error) {
    next(error);
  }
});

// Place a trading order
router.post('/orders', async (req, res, next) => {
  try {
    const { coin, is_buy, sz, limit_px, order_type, reduce_only } = req.body;
    
    // Validate required fields
    if (!coin || typeof is_buy !== 'boolean' || !sz || !limit_px) {
      return res.status(400).json({ 
        error: 'Missing required fields: coin, is_buy, sz, limit_px' 
      });
    }

    const orderParams = {
      coin,
      is_buy,
      sz: parseFloat(sz),
      limit_px: parseFloat(limit_px),
      order_type: order_type || { limit: { tif: 'Gtc' } },
      reduce_only: reduce_only || false
    };

    const result = await hyperliquidService.placeOrder(orderParams);
    res.json({ success: true, data: result });
    
  } catch (error) {
    next(error);
  }
});

// Cancel an order
router.delete('/orders/:orderId', async (req, res, next) => {
  try {
    const { orderId } = req.params;
    const { coin } = req.body;
    
    if (!coin) {
      return res.status(400).json({ error: 'Missing required field: coin' });
    }

    const result = await hyperliquidService.cancelOrder({
      coin,
      o: parseInt(orderId)
    });
    
    res.json({ success: true, data: result });
    
  } catch (error) {
    next(error);
  }
});

module.exports = router;
```

### Health Check Routes

```javascript
// src/routes/health.js
const express = require('express');
const router = express.Router();
const databaseService = require('../services/database');
const hyperliquidService = require('../services/hyperliquid');

router.get('/', async (req, res) => {
  const health = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    services: {}
  };

  try {
    // Check database connections
    if (databaseService.postgres) {
      await databaseService.query('SELECT 1');
      health.services.postgresql = 'healthy';
    }
    
    if (databaseService.mongodb) {
      await databaseService.mongodb.connection.db.admin().ping();
      health.services.mongodb = 'healthy';
    }

    // Check Hyperliquid connection
    health.services.hyperliquid = hyperliquidService.connected ? 'healthy' : 'disconnected';

    res.json(health);
    
  } catch (error) {
    health.status = 'unhealthy';
    health.error = error.message;
    res.status(503).json(health);
  }
});

router.get('/detailed', async (req, res) => {
  const details = {
    uptime: process.uptime(),
    memory: process.memoryUsage(),
    cpu: process.cpuUsage(),
    platform: process.platform,
    nodeVersion: process.version,
    environment: process.env.NODE_ENV
  };

  res.json(details);
});

module.exports = router;
```

## WebSocket Implementation

### Real-time Price Updates

```javascript
// src/services/websocket.js
const WebSocket = require('ws');
const hyperliquidService = require('./hyperliquid');
const logger = require('../utils/logger');

class WebSocketServer {
  constructor(server) {
    this.wss = new WebSocket.Server({ server });
    this.clients = new Map();
    this.priceSubscriptions = new Set();
    
    this.setupServer();
    this.setupHyperliquidSubscriptions();
  }

  setupServer() {
    this.wss.on('connection', (ws, req) => {
      const clientId = this.generateClientId();
      this.clients.set(clientId, {
        ws,
        subscriptions: new Set(),
        lastPing: Date.now()
      });

      logger.info('WebSocket client connected', { clientId, ip: req.socket.remoteAddress });

      ws.on('message', (data) => {
        this.handleMessage(clientId, data);
      });

      ws.on('close', () => {
        this.handleDisconnect(clientId);
      });

      ws.on('pong', () => {
        const client = this.clients.get(clientId);
        if (client) {
          client.lastPing = Date.now();
        }
      });

      // Send welcome message
      this.sendToClient(clientId, {
        type: 'welcome',
        clientId,
        timestamp: Date.now()
      });
    });

    // Heartbeat to detect broken connections
    setInterval(() => {
      this.wss.clients.forEach((ws) => {
        const clientId = this.findClientId(ws);
        const client = this.clients.get(clientId);
        
        if (client && Date.now() - client.lastPing > 30000) {
          ws.terminate();
          this.clients.delete(clientId);
        } else {
          ws.ping();
        }
      });
    }, 30000);
  }

  setupHyperliquidSubscriptions() {
    // Subscribe to price updates
    hyperliquidService.subscribeToAllMids((data) => {
      this.broadcastToSubscribers('prices', {
        type: 'priceUpdate',
        data,
        timestamp: Date.now()
      });
    });
  }

  handleMessage(clientId, data) {
    try {
      const message = JSON.parse(data);
      const client = this.clients.get(clientId);

      if (!client) return;

      switch (message.type) {
        case 'subscribe':
          this.handleSubscribe(clientId, message.channel);
          break;
        case 'unsubscribe':
          this.handleUnsubscribe(clientId, message.channel);
          break;
        case 'ping':
          this.sendToClient(clientId, { type: 'pong', timestamp: Date.now() });
          break;
        default:
          logger.warn('Unknown message type', { type: message.type, clientId });
      }
    } catch (error) {
      logger.error('Error handling WebSocket message', { error: error.message, clientId });
    }
  }

  handleSubscribe(clientId, channel) {
    const client = this.clients.get(clientId);
    if (!client) return;

    client.subscriptions.add(channel);
    
    if (channel === 'prices') {
      this.priceSubscriptions.add(clientId);
    }

    this.sendToClient(clientId, {
      type: 'subscribed',
      channel,
      timestamp: Date.now()
    });

    logger.debug('Client subscribed', { clientId, channel });
  }

  handleUnsubscribe(clientId, channel) {
    const client = this.clients.get(clientId);
    if (!client) return;

    client.subscriptions.delete(channel);
    
    if (channel === 'prices') {
      this.priceSubscriptions.delete(clientId);
    }

    this.sendToClient(clientId, {
      type: 'unsubscribed',
      channel,
      timestamp: Date.now()
    });

    logger.debug('Client unsubscribed', { clientId, channel });
  }

  handleDisconnect(clientId) {
    this.priceSubscriptions.delete(clientId);
    this.clients.delete(clientId);
    logger.info('WebSocket client disconnected', { clientId });
  }

  sendToClient(clientId, message) {
    const client = this.clients.get(clientId);
    if (client && client.ws.readyState === WebSocket.OPEN) {
      client.ws.send(JSON.stringify(message));
    }
  }

  broadcastToSubscribers(channel, message) {
    const subscribers = channel === 'prices' ? this.priceSubscriptions : [];
    
    subscribers.forEach(clientId => {
      this.sendToClient(clientId, message);
    });
  }

  generateClientId() {
    return Math.random().toString(36).substring(2) + Date.now().toString(36);
  }

  findClientId(ws) {
    for (const [clientId, client] of this.clients) {
      if (client.ws === ws) {
        return clientId;
      }
    }
    return null;
  }
}

module.exports = WebSocketServer;
```

## Error Handling

### Global Error Handler

```javascript
// src/middleware/errorHandler.js
const logger = require('../utils/logger');

class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}

const errorHandler = (err, req, res, next) => {
  let error = { ...err };
  error.message = err.message;

  // Log error
  logger.error('Error occurred', {
    error: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method,
    ip: req.ip
  });

  // Mongoose bad ObjectId
  if (err.name === 'CastError') {
    const message = 'Resource not found';
    error = new AppError(message, 404);
  }

  // Mongoose duplicate key
  if (err.code === 11000) {
    const message = 'Duplicate field value entered';
    error = new AppError(message, 400);
  }

  // Mongoose validation error
  if (err.name === 'ValidationError') {
    const message = Object.values(err.errors).map(val => val.message).join(', ');
    error = new AppError(message, 400);
  }

  // JWT errors
  if (err.name === 'JsonWebTokenError') {
    const message = 'Invalid token';
    error = new AppError(message, 401);
  }

  if (err.name === 'TokenExpiredError') {
    const message = 'Token expired';
    error = new AppError(message, 401);
  }

  res.status(error.statusCode || 500).json({
    success: false,
    error: error.message || 'Server Error',
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
  });
};

module.exports = { errorHandler, AppError };
```

### Async Error Wrapper

```javascript
// src/utils/asyncHandler.js
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

module.exports = asyncHandler;

// Usage example:
const asyncHandler = require('../utils/asyncHandler');

router.get('/users', asyncHandler(async (req, res) => {
  const users = await User.find();
  res.json({ success: true, data: users });
}));
```

## Security

### Authentication Middleware

```javascript
// src/middleware/auth.js
const jwt = require('jsonwebtoken');
const config = require('../config');
const { AppError } = require('./errorHandler');
const User = require('../models/User');

const authenticate = async (req, res, next) => {
  try {
    let token;

    if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
      token = req.headers.authorization.split(' ')[1];
    }

    if (!token) {
      return next(new AppError('Not authorized to access this route', 401));
    }

    const decoded = jwt.verify(token, config.jwt.secret);
    const user = await User.findById(decoded.id);

    if (!user) {
      return next(new AppError('No user found with this id', 401));
    }

    req.user = user;
    next();
  } catch (error) {
    return next(new AppError('Not authorized to access this route', 401));
  }
};

// Optional authentication (doesn't throw error if no token)
const optionalAuth = async (req, res, next) => {
  try {
    let token;

    if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
      token = req.headers.authorization.split(' ')[1];
      const decoded = jwt.verify(token, config.jwt.secret);
      const user = await User.findById(decoded.id);
      req.user = user;
    }

    next();
  } catch (error) {
    next();
  }
};

module.exports = { authenticate, optionalAuth };
```

### Input Validation

```javascript
// src/utils/validation.js
const validator = require('validator');

class ValidationError extends Error {
  constructor(message) {
    super(message);
    this.name = 'ValidationError';
  }
}

const validateWalletAddress = (address) => {
  if (!address || typeof address !== 'string') {
    throw new ValidationError('Wallet address is required');
  }
  
  if (!/^0x[a-fA-F0-9]{40}$/.test(address)) {
    throw new ValidationError('Invalid wallet address format');
  }
  
  return address.toLowerCase();
};

const validateOrderParams = (params) => {
  const { coin, is_buy, sz, limit_px } = params;
  
  if (!coin || typeof coin !== 'string') {
    throw new ValidationError('Coin symbol is required');
  }
  
  if (typeof is_buy !== 'boolean') {
    throw new ValidationError('Order side (is_buy) must be boolean');
  }
  
  if (!sz || isNaN(parseFloat(sz)) || parseFloat(sz) <= 0) {
    throw new ValidationError('Invalid size amount');
  }
  
  if (!limit_px || isNaN(parseFloat(limit_px)) || parseFloat(limit_px) <= 0) {
    throw new ValidationError('Invalid limit price');
  }
  
  return {
    coin: coin.toUpperCase(),
    is_buy,
    sz: parseFloat(sz),
    limit_px: parseFloat(limit_px)
  };
};

const validateEmail = (email) => {
  if (!email || !validator.isEmail(email)) {
    throw new ValidationError('Valid email is required');
  }
  return email.toLowerCase();
};

module.exports = {
  ValidationError,
  validateWalletAddress,
  validateOrderParams,
  validateEmail
};
```

## Performance & Optimization

### Caching Layer

```javascript
// src/services/cache.js
const NodeCache = require('node-cache');
const Redis = require('redis');
const config = require('../config');
const logger = require('../utils/logger');

class CacheService {
  constructor() {
    // In-memory cache for small, frequently accessed data
    this.memoryCache = new NodeCache({ 
      stdTTL: 300, // 5 minutes default
      checkperiod: 120 
    });

    // Redis for larger, shared cache
    if (config.redis?.url) {
      this.redisClient = Redis.createClient({ url: config.redis.url });
      this.redisClient.on('error', (err) => {
        logger.error('Redis error:', err);
      });
    }
  }

  async get(key, useRedis = false) {
    if (useRedis && this.redisClient) {
      try {
        const value = await this.redisClient.get(key);
        return value ? JSON.parse(value) : null;
      } catch (error) {
        logger.error('Redis get error:', error);
        return null;
      }
    }
    
    return this.memoryCache.get(key);
  }

  async set(key, value, ttl = 300, useRedis = false) {
    if (useRedis && this.redisClient) {
      try {
        await this.redisClient.setEx(key, ttl, JSON.stringify(value));
      } catch (error) {
        logger.error('Redis set error:', error);
      }
    } else {
      this.memoryCache.set(key, value, ttl);
    }
  }

  async del(key, useRedis = false) {
    if (useRedis && this.redisClient) {
      try {
        await this.redisClient.del(key);
      } catch (error) {
        logger.error('Redis delete error:', error);
      }
    } else {
      this.memoryCache.del(key);
    }
  }

  // Cache wrapper for functions
  async cached(key, fn, ttl = 300, useRedis = false) {
    let result = await this.get(key, useRedis);
    
    if (result === undefined || result === null) {
      result = await fn();
      if (result !== undefined && result !== null) {
        await this.set(key, result, ttl, useRedis);
      }
    }
    
    return result;
  }

  async close() {
    if (this.redisClient) {
      await this.redisClient.quit();
    }
    this.memoryCache.close();
  }
}

module.exports = new CacheService();
```

### Rate Limiting

```javascript
// src/middleware/rateLimit.js
const rateLimit = require('express-rate-limit');
const slowDown = require('express-slow-down');
const MongoStore = require('rate-limit-mongo');

// General API rate limiting
const createRateLimit = (windowMs, max, message) => rateLimit({
  store: new MongoStore({
    uri: process.env.MONGODB_URI,
    collectionName: 'rate_limits',
    expireTimeMs: windowMs
  }),
  windowMs,
  max,
  message: { error: message },
  standardHeaders: true,
  legacyHeaders: false,
  handler: (req, res) => {
    logger.warn('Rate limit exceeded', {
      ip: req.ip,
      path: req.path,
      userAgent: req.get('User-Agent')
    });
    res.status(429).json({ error: message });
  }
});

// Different limits for different endpoints
const apiLimiter = createRateLimit(
  15 * 60 * 1000, // 15 minutes
  100, // requests per window
  'Too many API requests, please try again later'
);

const tradingLimiter = createRateLimit(
  60 * 1000, // 1 minute
  10, // requests per window
  'Too many trading requests, please slow down'
);

const authLimiter = createRateLimit(
  15 * 60 * 1000, // 15 minutes
  5, // requests per window
  'Too many authentication attempts, please try again later'
);

// Slow down repeated requests instead of blocking
const speedLimiter = slowDown({
  windowMs: 15 * 60 * 1000, // 15 minutes
  delayAfter: 50, // allow 50 requests per windowMs without delay
  delayMs: 500 // add 500ms delay per request after delayAfter
});

module.exports = {
  apiLimiter,
  tradingLimiter,
  authLimiter,
  speedLimiter
};
```

## Testing

### Unit Testing Setup

```javascript
// tests/setup.js
const mongoose = require('mongoose');
const { MongoMemoryServer } = require('mongodb-memory-server');

let mongoServer;

beforeAll(async () => {
  mongoServer = await MongoMemoryServer.create();
  const mongoUri = mongoServer.getUri();
  await mongoose.connect(mongoUri);
});

afterAll(async () => {
  await mongoose.disconnect();
  await mongoServer.stop();
});

afterEach(async () => {
  const collections = mongoose.connection.collections;
  for (const key in collections) {
    await collections[key].deleteMany({});
  }
});
```

### API Testing

```javascript
// tests/api.test.js
const request = require('supertest');
const app = require('../src/app');
const User = require('../src/models/User');

describe('API Routes', () => {
  let authToken;
  let testUser;

  beforeEach(async () => {
    testUser = await User.create({
      username: 'testuser',
      email: 'test@example.com',
      password: 'hashedpassword'
    });
    
    authToken = jwt.sign({ id: testUser._id }, process.env.JWT_SECRET);
  });

  describe('GET /api/prices', () => {
    it('should return current market prices', async () => {
      const res = await request(app)
        .get('/api/prices')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);

      expect(res.body.success).toBe(true);
      expect(res.body.data).toBeDefined();
    });

    it('should fail without authentication', async () => {
      await request(app)
        .get('/api/prices')
        .expect(401);
    });
  });

  describe('POST /api/orders', () => {
    it('should place a valid order', async () => {
      const orderData = {
        coin: 'BTC',
        is_buy: true,
        sz: '0.1',
        limit_px: '50000'
      };

      const res = await request(app)
        .post('/api/orders')
        .set('Authorization', `Bearer ${authToken}`)
        .send(orderData)
        .expect(200);

      expect(res.body.success).toBe(true);
    });

    it('should reject invalid order data', async () => {
      const invalidOrder = {
        coin: 'BTC',
        is_buy: true
        // missing sz and limit_px
      };

      await request(app)
        .post('/api/orders')
        .set('Authorization', `Bearer ${authToken}`)
        .send(invalidOrder)
        .expect(400);
    });
  });
});
```

## Deployment

### Production Setup

```javascript
// src/server.js
const app = require('./app');
const config = require('./config');
const logger = require('./utils/logger');
const databaseService = require('./services/database');
const hyperliquidService = require('./services/hyperliquid');
const WebSocketServer = require('./services/websocket');

const server = require('http').createServer(app);

async function startServer() {
  try {
    // Initialize services
    await databaseService.initialize();
    await hyperliquidService.initialize();
    
    // Start WebSocket server
    new WebSocketServer(server);
    
    // Start HTTP server
    server.listen(config.port, () => {
      logger.info(`Server running on port ${config.port} in ${config.env} mode`);
    });

    // Graceful shutdown
    const gracefulShutdown = async (signal) => {
      logger.info(`Received ${signal}. Starting graceful shutdown...`);
      
      server.close(async () => {
        try {
          await hyperliquidService.disconnect();
          await databaseService.close();
          logger.info('Graceful shutdown completed');
          process.exit(0);
        } catch (error) {
          logger.error('Error during shutdown:', error);
          process.exit(1);
        }
      });
    };

    process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
    process.on('SIGINT', () => gracefulShutdown('SIGINT'));

  } catch (error) {
    logger.error('Failed to start server:', error);
    process.exit(1);
  }
}

startServer();
```

### Process Management (PM2)

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'hyperliquid-telegram-bot',
    script: 'src/server.js',
    instances: 'max',
    exec_mode: 'cluster',
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    },
    error_file: './logs/err.log',
    out_file: './logs/out.log',
    log_file: './logs/combined.log',
    time: true,
    max_memory_restart: '1G',
    node_args: '--max-old-space-size=1024'
  }]
};
```

### Docker Configuration

```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY src/ ./src/
COPY config/ ./config/

# Create logs directory
RUN mkdir -p logs

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodeuser -u 1001
USER nodeuser

EXPOSE 3000

CMD ["node", "src/server.js"]
```

## Best Practices Summary

1. **Environment Configuration**: Use environment variables for all configuration
2. **Error Handling**: Implement comprehensive error handling and logging
3. **Security**: Use authentication, rate limiting, and input validation
4. **Performance**: Implement caching and connection pooling
5. **Testing**: Write comprehensive unit and integration tests
6. **Monitoring**: Implement health checks and performance monitoring
7. **Documentation**: Document your API and code thoroughly
8. **Deployment**: Use process managers and containerization for production
9. **Graceful Shutdown**: Handle shutdown signals properly
10. **Code Quality**: Use linting, formatting, and code review processes

## Resources

- [Node.js Documentation](https://nodejs.org/docs/)
- [Express.js Guide](https://expressjs.com/en/guide/)
- [PM2 Documentation](https://pm2.keymetrics.io/docs/)
- [Jest Testing Framework](https://jestjs.io/docs/)
- [Winston Logger](https://github.com/winstonjs/winston) 