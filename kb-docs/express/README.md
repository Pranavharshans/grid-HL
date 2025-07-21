# Express.js Documentation

## Overview

Express.js is a minimal and flexible Node.js web application framework that provides a robust set of features for web and mobile applications. This documentation covers building REST API endpoints for the Hyperliquid Telegram bot backend.

## Table of Contents

1. [Installation & Setup](#installation--setup)
2. [Basic Application Structure](#basic-application-structure)
3. [Middleware](#middleware)
4. [Routing](#routing)
5. [Request & Response Handling](#request--response-handling)
6. [API Design Patterns](#api-design-patterns)
7. [Authentication & Authorization](#authentication--authorization)
8. [Error Handling](#error-handling)
9. [Testing Express APIs](#testing-express-apis)
10. [Production Considerations](#production-considerations)

## Installation & Setup

```bash
npm install express
npm install --save-dev @types/express  # For TypeScript
```

### Basic Express Server

```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

// Basic middleware
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Basic route
app.get('/', (req, res) => {
  res.json({ message: 'Hyperliquid Telegram Bot API' });
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

## Basic Application Structure

### Modular Application Setup

```javascript
// app.js
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const compression = require('compression');
const morgan = require('morgan');

const apiRoutes = require('./routes/api');
const webhookRoutes = require('./routes/webhook');
const errorHandler = require('./middleware/errorHandler');
const notFound = require('./middleware/notFound');

const app = express();

// Security middleware
app.use(helmet());
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  credentials: true
}));

// Performance middleware
app.use(compression());

// Logging middleware
app.use(morgan('combined'));

// Body parsing middleware
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Static files (if needed)
app.use(express.static('public'));

// API routes
app.use('/api/v1', apiRoutes);
app.use('/webhook', webhookRoutes);

// Error handling
app.use(notFound);
app.use(errorHandler);

module.exports = app;
```

### Environment Configuration

```javascript
// config/express.js
const express = require('express');
const config = require('./index');

module.exports = (app) => {
  // Environment-specific configuration
  if (config.env === 'development') {
    app.use(require('morgan')('dev'));
    app.set('json spaces', 2); // Pretty print JSON in dev
  }

  if (config.env === 'production') {
    app.use(require('morgan')('combined'));
    app.set('trust proxy', 1); // Trust first proxy
  }

  // Global middleware
  app.use(express.json({ 
    limit: config.bodyParser.limit,
    verify: (req, res, buf) => {
      req.rawBody = buf; // Store raw body for webhook verification
    }
  }));

  app.use(express.urlencoded({ 
    extended: true,
    limit: config.bodyParser.limit
  }));

  return app;
};
```

## Middleware

### Custom Middleware Functions

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const { promisify } = require('util');

const authenticateToken = async (req, res, next) => {
  try {
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1];

    if (!token) {
      return res.status(401).json({ error: 'Access token required' });
    }

    const decoded = await promisify(jwt.verify)(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(403).json({ error: 'Invalid or expired token' });
  }
};

const optionalAuth = async (req, res, next) => {
  try {
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1];

    if (token) {
      const decoded = await promisify(jwt.verify)(token, process.env.JWT_SECRET);
      req.user = decoded;
    }
  } catch (error) {
    // Continue without authentication
  }
  next();
};

module.exports = { authenticateToken, optionalAuth };
```

### Request Validation Middleware

```javascript
// middleware/validation.js
const { body, param, query, validationResult } = require('express-validator');

const handleValidationErrors = (req, res, next) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({
      error: 'Validation failed',
      details: errors.array()
    });
  }
  next();
};

// Validation rules
const validateOrder = [
  body('coin').notEmpty().withMessage('Coin symbol is required'),
  body('is_buy').isBoolean().withMessage('Order side must be boolean'),
  body('sz').isFloat({ min: 0.000001 }).withMessage('Size must be positive'),
  body('limit_px').isFloat({ min: 0.01 }).withMessage('Price must be positive'),
  handleValidationErrors
];

const validateWalletAddress = [
  param('address').matches(/^0x[a-fA-F0-9]{40}$/).withMessage('Invalid wallet address'),
  handleValidationErrors
];

const validatePagination = [
  query('page').optional().isInt({ min: 1 }).withMessage('Page must be positive integer'),
  query('limit').optional().isInt({ min: 1, max: 100 }).withMessage('Limit must be 1-100'),
  handleValidationErrors
];

module.exports = {
  validateOrder,
  validateWalletAddress,
  validatePagination,
  handleValidationErrors
};
```

### Rate Limiting Middleware

```javascript
// middleware/rateLimiter.js
const rateLimit = require('express-rate-limit');
const slowDown = require('express-slow-down');

// General API rate limiter
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: {
    error: 'Too many requests from this IP, please try again later.',
    retryAfter: Math.ceil(15 * 60)
  },
  standardHeaders: true,
  legacyHeaders: false,
});

// Strict limiter for trading endpoints
const tradingLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 10, // limit each IP to 10 trading requests per minute
  message: {
    error: 'Too many trading requests, please slow down.',
    retryAfter: 60
  }
});

// Authentication endpoints limiter
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // limit each IP to 5 auth requests per 15 minutes
  message: {
    error: 'Too many authentication attempts, please try again later.',
    retryAfter: Math.ceil(15 * 60)
  }
});

// Speed limiter - slow down instead of blocking
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

## Routing

### RESTful API Routes

```javascript
// routes/api.js
const express = require('express');
const router = express.Router();
const { authenticateToken } = require('../middleware/auth');
const { validateOrder, validateWalletAddress } = require('../middleware/validation');
const { tradingLimiter } = require('../middleware/rateLimiter');
const hyperliquidController = require('../controllers/hyperliquid');
const userController = require('../controllers/user');

// Public routes
router.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

router.get('/prices', hyperliquidController.getAllPrices);
router.get('/markets', hyperliquidController.getMarkets);

// Protected routes
router.use(authenticateToken);

// User routes
router.get('/user/profile', userController.getProfile);
router.put('/user/profile', userController.updateProfile);
router.post('/user/wallet', validateWalletAddress, userController.addWallet);

// Account routes
router.get('/account/:address', validateWalletAddress, hyperliquidController.getAccountState);
router.get('/account/:address/orders', validateWalletAddress, hyperliquidController.getOpenOrders);
router.get('/account/:address/trades', validateWalletAddress, hyperliquidController.getTradeHistory);

// Trading routes (with rate limiting)
router.use('/orders', tradingLimiter);
router.post('/orders', validateOrder, hyperliquidController.placeOrder);
router.delete('/orders/:orderId', hyperliquidController.cancelOrder);
router.delete('/orders', hyperliquidController.cancelAllOrders);

// Market data routes
router.get('/orderbook/:symbol', hyperliquidController.getOrderBook);
router.get('/candles/:symbol', hyperliquidController.getCandleData);

module.exports = router;
```

### Webhook Routes

```javascript
// routes/webhook.js
const express = require('express');
const router = express.Router();
const crypto = require('crypto');
const telegramController = require('../controllers/telegram');

// Telegram webhook verification middleware
const verifyTelegramWebhook = (req, res, next) => {
  const token = process.env.BOT_TOKEN;
  const secretKey = crypto.createHash('sha256').update(token).digest();
  
  const signature = req.headers['x-telegram-bot-api-secret-token'];
  const expectedSignature = crypto
    .createHmac('sha256', secretKey)
    .update(req.rawBody)
    .digest('hex');

  if (signature !== expectedSignature) {
    return res.status(403).json({ error: 'Unauthorized' });
  }

  next();
};

// Telegram webhook endpoint
router.post('/telegram', verifyTelegramWebhook, telegramController.handleUpdate);

// Health check for webhooks
router.get('/telegram/health', (req, res) => {
  res.json({ 
    status: 'ready',
    webhook: true,
    timestamp: new Date().toISOString()
  });
});

module.exports = router;
```

### Route Parameters & Middleware

```javascript
// Advanced routing patterns
const router = express.Router();

// Route parameters with validation
router.param('userId', async (req, res, next, id) => {
  try {
    if (!mongoose.Types.ObjectId.isValid(id)) {
      return res.status(400).json({ error: 'Invalid user ID' });
    }
    
    const user = await User.findById(id);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    req.user = user;
    next();
  } catch (error) {
    next(error);
  }
});

// Nested routes
router.get('/users/:userId', (req, res) => {
  res.json(req.user); // User already loaded by param middleware
});

router.get('/users/:userId/trades', async (req, res) => {
  const trades = await Trade.find({ userId: req.params.userId });
  res.json(trades);
});

// Route-specific middleware
const adminOnly = (req, res, next) => {
  if (!req.user.isAdmin) {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

router.delete('/users/:userId', adminOnly, userController.deleteUser);
```

## Request & Response Handling

### Request Processing

```javascript
// controllers/hyperliquid.js
const hyperliquidService = require('../services/hyperliquid');
const cacheService = require('../services/cache');
const logger = require('../utils/logger');

class HyperliquidController {
  // Get all market prices with caching
  async getAllPrices(req, res, next) {
    try {
      const cacheKey = 'market:prices:all';
      
      let prices = await cacheService.get(cacheKey);
      
      if (!prices) {
        prices = await hyperliquidService.getAllMids();
        await cacheService.set(cacheKey, prices, 10); // Cache for 10 seconds
      }
      
      res.json({
        success: true,
        data: prices,
        cached: !!prices.cached,
        timestamp: new Date().toISOString()
      });
    } catch (error) {
      next(error);
    }
  }

  // Place trading order
  async placeOrder(req, res, next) {
    try {
      const { coin, is_buy, sz, limit_px, order_type, reduce_only } = req.body;
      const userId = req.user.id;
      
      // Log the trading attempt
      logger.info('Trade order attempt', {
        userId,
        coin,
        is_buy,
        sz,
        limit_px,
        ip: req.ip
      });

      const orderParams = {
        coin: coin.toUpperCase(),
        is_buy,
        sz: parseFloat(sz),
        limit_px: parseFloat(limit_px),
        order_type: order_type || { limit: { tif: 'Gtc' } },
        reduce_only: reduce_only || false
      };

      const result = await hyperliquidService.placeOrder(orderParams);
      
      // Save trade to database
      await Trade.create({
        userId,
        ...orderParams,
        orderId: result.orderId,
        status: 'pending'
      });

      res.status(201).json({
        success: true,
        data: result,
        message: 'Order placed successfully'
      });
      
    } catch (error) {
      logger.error('Order placement failed', {
        userId: req.user.id,
        error: error.message,
        body: req.body
      });
      next(error);
    }
  }

  // Get account state with pagination
  async getAccountState(req, res, next) {
    try {
      const { address } = req.params;
      const page = parseInt(req.query.page) || 1;
      const limit = parseInt(req.query.limit) || 20;
      
      const cacheKey = `account:${address}:state`;
      let accountState = await cacheService.get(cacheKey);
      
      if (!accountState) {
        accountState = await hyperliquidService.getAccountState(address);
        await cacheService.set(cacheKey, accountState, 30); // Cache for 30 seconds
      }

      // Paginate positions if they exist
      if (accountState.positions) {
        const startIndex = (page - 1) * limit;
        const endIndex = startIndex + limit;
        const paginatedPositions = accountState.positions.slice(startIndex, endIndex);
        
        accountState = {
          ...accountState,
          positions: paginatedPositions,
          pagination: {
            page,
            limit,
            total: accountState.positions.length,
            pages: Math.ceil(accountState.positions.length / limit)
          }
        };
      }

      res.json({
        success: true,
        data: accountState
      });
      
    } catch (error) {
      next(error);
    }
  }
}

module.exports = new HyperliquidController();
```

### Response Formatting

```javascript
// utils/response.js
class ResponseFormatter {
  static success(data, message = 'Success', meta = {}) {
    return {
      success: true,
      message,
      data,
      meta: {
        timestamp: new Date().toISOString(),
        ...meta
      }
    };
  }

  static error(message, code = 'INTERNAL_ERROR', details = null) {
    return {
      success: false,
      error: {
        message,
        code,
        details,
        timestamp: new Date().toISOString()
      }
    };
  }

  static paginated(data, pagination) {
    return {
      success: true,
      data,
      pagination: {
        page: pagination.page,
        limit: pagination.limit,
        total: pagination.total,
        pages: Math.ceil(pagination.total / pagination.limit)
      },
      meta: {
        timestamp: new Date().toISOString()
      }
    };
  }
}

// Usage in controllers
app.get('/api/users', async (req, res) => {
  const users = await User.find();
  res.json(ResponseFormatter.success(users, 'Users retrieved successfully'));
});

app.use((error, req, res, next) => {
  res.status(error.statusCode || 500).json(
    ResponseFormatter.error(error.message, error.code)
  );
});
```

## API Design Patterns

### RESTful Resource Design

```javascript
// Consistent RESTful patterns
const router = express.Router();

// GET /api/users - Get all users
router.get('/users', userController.getAll);

// GET /api/users/:id - Get specific user
router.get('/users/:id', userController.getById);

// POST /api/users - Create new user
router.post('/users', userController.create);

// PUT /api/users/:id - Update entire user
router.put('/users/:id', userController.update);

// PATCH /api/users/:id - Partial update
router.patch('/users/:id', userController.partialUpdate);

// DELETE /api/users/:id - Delete user
router.delete('/users/:id', userController.delete);

// Nested resources
// GET /api/users/:id/trades - Get user's trades
router.get('/users/:id/trades', tradeController.getByUser);

// POST /api/users/:id/trades - Create trade for user
router.post('/users/:id/trades', tradeController.createForUser);
```

### API Versioning

```javascript
// Version-specific routes
// routes/v1/index.js
const express = require('express');
const router = express.Router();

router.use('/users', require('./users'));
router.use('/trades', require('./trades'));
router.use('/hyperliquid', require('./hyperliquid'));

module.exports = router;

// routes/v2/index.js (future version)
const express = require('express');
const router = express.Router();

router.use('/users', require('./users'));
router.use('/orders', require('./orders')); // Different structure
router.use('/portfolio', require('./portfolio')); // New endpoints

module.exports = router;

// Main app.js
app.use('/api/v1', require('./routes/v1'));
app.use('/api/v2', require('./routes/v2'));

// Header-based versioning alternative
const versionMiddleware = (req, res, next) => {
  const version = req.headers['api-version'] || 'v1';
  req.apiVersion = version;
  next();
};

app.use('/api', versionMiddleware);
```

### Query Parameter Handling

```javascript
// controllers/baseController.js
class BaseController {
  parseQueryParams(req) {
    const {
      page = 1,
      limit = 20,
      sort = 'createdAt',
      order = 'desc',
      search,
      ...filters
    } = req.query;

    return {
      pagination: {
        page: Math.max(1, parseInt(page)),
        limit: Math.min(100, Math.max(1, parseInt(limit)))
      },
      sorting: {
        field: sort,
        direction: order === 'asc' ? 1 : -1
      },
      search,
      filters: this.sanitizeFilters(filters)
    };
  }

  sanitizeFilters(filters) {
    const allowed = ['status', 'symbol', 'type', 'createdAt'];
    return Object.keys(filters)
      .filter(key => allowed.includes(key))
      .reduce((obj, key) => {
        obj[key] = filters[key];
        return obj;
      }, {});
  }

  buildMongoQuery(params) {
    const query = {};

    // Add filters
    Object.entries(params.filters).forEach(([key, value]) => {
      if (key === 'createdAt' && value.includes('|')) {
        const [start, end] = value.split('|');
        query[key] = { $gte: new Date(start), $lte: new Date(end) };
      } else {
        query[key] = { $regex: value, $options: 'i' };
      }
    });

    // Add search
    if (params.search) {
      query.$text = { $search: params.search };
    }

    return query;
  }
}
```

## Authentication & Authorization

### JWT Implementation

```javascript
// services/auth.js
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');
const User = require('../models/User');

class AuthService {
  generateTokens(user) {
    const payload = {
      id: user._id,
      username: user.username,
      role: user.role
    };

    const accessToken = jwt.sign(payload, process.env.JWT_SECRET, {
      expiresIn: '15m'
    });

    const refreshToken = jwt.sign(payload, process.env.JWT_REFRESH_SECRET, {
      expiresIn: '7d'
    });

    return { accessToken, refreshToken };
  }

  async verifyToken(token, isRefresh = false) {
    const secret = isRefresh ? process.env.JWT_REFRESH_SECRET : process.env.JWT_SECRET;
    return jwt.verify(token, secret);
  }

  async hashPassword(password) {
    return bcrypt.hash(password, 12);
  }

  async comparePassword(password, hash) {
    return bcrypt.compare(password, hash);
  }

  async login(username, password) {
    const user = await User.findOne({ username }).select('+password');
    
    if (!user || !await this.comparePassword(password, user.password)) {
      throw new Error('Invalid credentials');
    }

    const tokens = this.generateTokens(user);
    
    // Save refresh token to database
    user.refreshToken = tokens.refreshToken;
    await user.save();

    return {
      user: user.toObject({ transform: (doc, ret) => delete ret.password && ret }),
      ...tokens
    };
  }

  async refreshToken(refreshToken) {
    const decoded = await this.verifyToken(refreshToken, true);
    const user = await User.findById(decoded.id);

    if (!user || user.refreshToken !== refreshToken) {
      throw new Error('Invalid refresh token');
    }

    return this.generateTokens(user);
  }

  async logout(userId) {
    await User.findByIdAndUpdate(userId, { refreshToken: null });
  }
}

module.exports = new AuthService();
```

### Role-Based Access Control

```javascript
// middleware/rbac.js
const authorize = (...roles) => {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ 
        error: 'Insufficient permissions',
        required: roles,
        current: req.user.role
      });
    }

    next();
  };
};

// Permission-based access control
const hasPermission = (permission) => {
  return async (req, res, next) => {
    try {
      const user = await User.findById(req.user.id).populate('role');
      
      if (!user.role.permissions.includes(permission)) {
        return res.status(403).json({ 
          error: 'Missing required permission',
          required: permission
        });
      }

      next();
    } catch (error) {
      next(error);
    }
  };
};

// Usage
router.delete('/users/:id', authorize('admin'), userController.delete);
router.post('/trades', hasPermission('trade'), tradeController.create);
```

## Error Handling

### Centralized Error Handling

```javascript
// middleware/errorHandler.js
const logger = require('../utils/logger');

class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}

const handleCastErrorDB = (err) => {
  const message = `Invalid ${err.path}: ${err.value}`;
  return new AppError(message, 400);
};

const handleDuplicateFieldsDB = (err) => {
  const value = err.errmsg.match(/(["'])(\\?.)*?\1/)[0];
  const message = `Duplicate field value: ${value}. Please use another value!`;
  return new AppError(message, 400);
};

const handleValidationErrorDB = (err) => {
  const errors = Object.values(err.errors).map(el => el.message);
  const message = `Invalid input data. ${errors.join('. ')}`;
  return new AppError(message, 400);
};

const handleJWTError = () =>
  new AppError('Invalid token. Please log in again!', 401);

const handleJWTExpiredError = () =>
  new AppError('Your token has expired! Please log in again.', 401);

const sendErrorDev = (err, res) => {
  res.status(err.statusCode).json({
    status: err.status,
    error: err,
    message: err.message,
    stack: err.stack
  });
};

const sendErrorProd = (err, res) => {
  // Operational, trusted error: send message to client
  if (err.isOperational) {
    res.status(err.statusCode).json({
      status: err.status,
      message: err.message
    });
  } else {
    // Programming or other unknown error: don't leak error details
    logger.error('ERROR:', err);

    res.status(500).json({
      status: 'error',
      message: 'Something went wrong!'
    });
  }
};

module.exports = (err, req, res, next) => {
  err.statusCode = err.statusCode || 500;
  err.status = err.status || 'error';

  if (process.env.NODE_ENV === 'development') {
    sendErrorDev(err, res);
  } else {
    let error = { ...err };
    error.message = err.message;

    if (error.name === 'CastError') error = handleCastErrorDB(error);
    if (error.code === 11000) error = handleDuplicateFieldsDB(error);
    if (error.name === 'ValidationError') error = handleValidationErrorDB(error);
    if (error.name === 'JsonWebTokenError') error = handleJWTError();
    if (error.name === 'TokenExpiredError') error = handleJWTExpiredError();

    sendErrorProd(error, res);
  }
};

module.exports.AppError = AppError;
```

### Async Error Wrapper

```javascript
// utils/catchAsync.js
const catchAsync = (fn) => {
  return (req, res, next) => {
    fn(req, res, next).catch(next);
  };
};

module.exports = catchAsync;

// Usage in controllers
const catchAsync = require('../utils/catchAsync');

exports.getUser = catchAsync(async (req, res, next) => {
  const user = await User.findById(req.params.id);
  
  if (!user) {
    return next(new AppError('No user found with that ID', 404));
  }

  res.status(200).json({
    status: 'success',
    data: { user }
  });
});
```

## Testing Express APIs

### Unit Testing Controllers

```javascript
// tests/controllers/user.test.js
const request = require('supertest');
const app = require('../../app');
const User = require('../../models/User');

describe('User Controller', () => {
  beforeEach(async () => {
    await User.deleteMany({});
  });

  describe('GET /api/v1/users', () => {
    it('should return all users', async () => {
      // Create test users
      await User.create([
        { username: 'user1', email: 'user1@test.com' },
        { username: 'user2', email: 'user2@test.com' }
      ]);

      const res = await request(app)
        .get('/api/v1/users')
        .expect(200);

      expect(res.body.success).toBe(true);
      expect(res.body.data).toHaveLength(2);
    });

    it('should handle pagination', async () => {
      // Create 25 test users
      const users = Array.from({ length: 25 }, (_, i) => ({
        username: `user${i}`,
        email: `user${i}@test.com`
      }));
      await User.create(users);

      const res = await request(app)
        .get('/api/v1/users?page=2&limit=10')
        .expect(200);

      expect(res.body.data).toHaveLength(10);
      expect(res.body.pagination.page).toBe(2);
      expect(res.body.pagination.total).toBe(25);
    });
  });

  describe('POST /api/v1/users', () => {
    it('should create a new user', async () => {
      const userData = {
        username: 'newuser',
        email: 'newuser@test.com',
        password: 'password123'
      };

      const res = await request(app)
        .post('/api/v1/users')
        .send(userData)
        .expect(201);

      expect(res.body.success).toBe(true);
      expect(res.body.data.username).toBe(userData.username);
      
      // Verify user was created in database
      const user = await User.findOne({ username: userData.username });
      expect(user).toBeTruthy();
    });

    it('should return validation errors for invalid data', async () => {
      const invalidData = {
        username: '', // Invalid: empty
        email: 'invalid-email', // Invalid: not email format
        password: '123' // Invalid: too short
      };

      const res = await request(app)
        .post('/api/v1/users')
        .send(invalidData)
        .expect(400);

      expect(res.body.success).toBe(false);
      expect(res.body.error.details).toHaveLength(3);
    });
  });
});
```

### Integration Testing

```javascript
// tests/integration/trading.test.js
const request = require('supertest');
const app = require('../../app');
const User = require('../../models/User');
const jwt = require('jsonwebtoken');

describe('Trading API Integration', () => {
  let authToken;
  let testUser;

  beforeEach(async () => {
    testUser = await User.create({
      username: 'trader',
      email: 'trader@test.com',
      password: 'hashedpassword'
    });

    authToken = jwt.sign(
      { id: testUser._id, username: testUser.username },
      process.env.JWT_SECRET
    );
  });

  describe('Complete trading flow', () => {
    it('should place and cancel an order', async () => {
      // Place order
      const orderData = {
        coin: 'BTC',
        is_buy: true,
        sz: '0.1',
        limit_px: '50000'
      };

      const placeRes = await request(app)
        .post('/api/v1/orders')
        .set('Authorization', `Bearer ${authToken}`)
        .send(orderData)
        .expect(201);

      expect(placeRes.body.success).toBe(true);
      const orderId = placeRes.body.data.orderId;

      // Cancel order
      const cancelRes = await request(app)
        .delete(`/api/v1/orders/${orderId}`)
        .set('Authorization', `Bearer ${authToken}`)
        .send({ coin: 'BTC' })
        .expect(200);

      expect(cancelRes.body.success).toBe(true);
    });
  });
});
```

## Production Considerations

### Security Headers

```javascript
// middleware/security.js
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');

const securityMiddleware = (app) => {
  // Helmet for security headers
  app.use(helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        scriptSrc: ["'self'"],
        imgSrc: ["'self'", "data:", "https:"],
      },
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true
    }
  }));

  // Rate limiting
  const limiter = rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 100,
    standardHeaders: true,
    legacyHeaders: false,
  });
  app.use(limiter);

  // Additional security headers
  app.use((req, res, next) => {
    res.setHeader('X-Content-Type-Options', 'nosniff');
    res.setHeader('X-Frame-Options', 'DENY');
    res.setHeader('X-XSS-Protection', '1; mode=block');
    next();
  });
};

module.exports = securityMiddleware;
```

### Performance Optimization

```javascript
// middleware/performance.js
const compression = require('compression');
const responseTime = require('response-time');

const performanceMiddleware = (app) => {
  // Gzip compression
  app.use(compression({
    filter: (req, res) => {
      if (req.headers['x-no-compression']) {
        return false;
      }
      return compression.filter(req, res);
    },
    level: 6,
    threshold: 1024
  }));

  // Response time header
  app.use(responseTime());

  // Cache control headers
  app.use('/api', (req, res, next) => {
    if (req.method === 'GET') {
      res.set('Cache-Control', 'public, max-age=60'); // 1 minute
    } else {
      res.set('Cache-Control', 'no-cache, no-store, must-revalidate');
    }
    next();
  });
};

module.exports = performanceMiddleware;
```

### Health Checks & Monitoring

```javascript
// routes/monitoring.js
const express = require('express');
const router = express.Router();
const os = require('os');

router.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    memory: process.memoryUsage(),
    load: os.loadavg()
  });
});

router.get('/metrics', (req, res) => {
  res.json({
    system: {
      platform: os.platform(),
      arch: os.arch(),
      cpus: os.cpus().length,
      memory: {
        total: os.totalmem(),
        free: os.freemem()
      }
    },
    process: {
      pid: process.pid,
      uptime: process.uptime(),
      memory: process.memoryUsage(),
      cpu: process.cpuUsage()
    }
  });
});

module.exports = router;
```

## Best Practices Summary

1. **Modular Structure**: Organize code into logical modules and use proper separation of concerns
2. **Middleware Order**: Apply middleware in the correct order (security, parsing, authentication, routes, error handling)
3. **Error Handling**: Implement centralized error handling with proper error types and status codes
4. **Validation**: Validate all inputs and sanitize data before processing
5. **Security**: Use security headers, rate limiting, and proper authentication
6. **Testing**: Write comprehensive unit and integration tests
7. **Documentation**: Document your API endpoints and maintain API versioning
8. **Performance**: Use compression, caching, and optimize database queries
9. **Monitoring**: Implement health checks and performance monitoring
10. **Production**: Configure for production with proper logging and error handling

## Resources

- [Express.js Official Documentation](https://expressjs.com/)
- [Express.js Security Best Practices](https://expressjs.com/en/advanced/best-practice-security.html)
- [RESTful API Design Guide](https://restfulapi.net/)
- [HTTP Status Codes](https://httpstatuses.com/) 