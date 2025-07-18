# Database Documentation

## Overview

This documentation covers database integration for Node.js applications using two popular database solutions: **MongoDB with Mongoose** and **PostgreSQL with node-postgres**. Both provide robust solutions for data persistence with different paradigms and use cases.

## Table of Contents

1. [MongoDB with Mongoose](#mongodb-with-mongoose)
2. [PostgreSQL with node-postgres](#postgresql-with-node-postgres)
3. [Database Design Patterns](#database-design-patterns)
4. [Performance Optimization](#performance-optimization)
5. [Error Handling](#error-handling)
6. [Security Best Practices](#security-best-practices)
7. [Migration Strategies](#migration-strategies)
8. [Monitoring & Maintenance](#monitoring--maintenance)

## MongoDB with Mongoose

### Installation & Setup

```bash
npm install mongoose
npm install @types/mongoose  # For TypeScript
```

### Basic Connection

```javascript
const mongoose = require('mongoose');

// Basic connection
async function connectDB() {
  try {
    await mongoose.connect('mongodb://127.0.0.1:27017/myapp');
    console.log('Connected to MongoDB');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
}

// With options
await mongoose.connect('mongodb://127.0.0.1:27017/myapp', {
  maxPoolSize: 10,
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
  bufferCommands: false,
  bufferMaxEntries: 0
});
```

### Schema Definition

```javascript
const { Schema, model } = require('mongoose');

// User Schema
const userSchema = new Schema({
  username: {
    type: String,
    required: [true, 'Username is required'],
    unique: true,
    trim: true,
    minlength: 3,
    maxlength: 30
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    match: [/^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/, 'Invalid email']
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  profile: {
    firstName: String,
    lastName: String,
    avatar: String,
    bio: { type: String, maxlength: 500 }
  },
  settings: {
    notifications: { type: Boolean, default: true },
    theme: { type: String, enum: ['light', 'dark'], default: 'light' }
  },
  trades: [{
    symbol: String,
    quantity: Number,
    price: Number,
    type: { type: String, enum: ['buy', 'sell'] },
    timestamp: { type: Date, default: Date.now }
  }],
  balance: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

// Pre-save middleware
userSchema.pre('save', function(next) {
  this.updatedAt = Date.now();
  next();
});

// Instance methods
userSchema.methods.getFullName = function() {
  return `${this.profile.firstName} ${this.profile.lastName}`;
};

// Static methods
userSchema.statics.findByEmail = function(email) {
  return this.findOne({ email: email.toLowerCase() });
};

// Virtual properties
userSchema.virtual('tradeCount').get(function() {
  return this.trades.length;
});

const User = model('User', userSchema);
```

### CRUD Operations

```javascript
// Create
const newUser = new User({
  username: 'trader123',
  email: 'trader@example.com',
  password: 'hashedPassword'
});
await newUser.save();

// Alternative create
const user = await User.create({
  username: 'trader456',
  email: 'trader2@example.com',
  password: 'hashedPassword'
});

// Read
const user = await User.findById(userId);
const userByEmail = await User.findByEmail('trader@example.com');
const users = await User.find({ 'settings.notifications': true });

// Query with population (if using refs)
const usersWithTrades = await User.find()
  .select('username email balance')
  .limit(10)
  .sort({ createdAt: -1 });

// Update
await User.findByIdAndUpdate(userId, {
  $set: { 'settings.theme': 'dark' },
  $inc: { balance: 100 }
}, { new: true });

// Update with validation
const updatedUser = await User.findByIdAndUpdate(
  userId,
  { username: 'newUsername' },
  { new: true, runValidators: true }
);

// Delete
await User.findByIdAndDelete(userId);
await User.deleteMany({ createdAt: { $lt: new Date('2023-01-01') } });
```

### Advanced Queries

```javascript
// Complex queries
const activeTraders = await User.find({
  balance: { $gte: 1000 },
  'trades.0': { $exists: true }, // Has at least one trade
  createdAt: { $gte: new Date('2024-01-01') }
}).select('username balance tradeCount');

// Aggregation pipeline
const tradeStats = await User.aggregate([
  { $match: { 'trades.0': { $exists: true } } },
  { $unwind: '$trades' },
  {
    $group: {
      _id: '$trades.symbol',
      totalVolume: { $sum: { $multiply: ['$trades.quantity', '$trades.price'] } },
      avgPrice: { $avg: '$trades.price' },
      tradeCount: { $sum: 1 }
    }
  },
  { $sort: { totalVolume: -1 } }
]);

// Text search (requires text index)
userSchema.index({ username: 'text', 'profile.bio': 'text' });
const searchResults = await User.find(
  { $text: { $search: 'crypto trading' } },
  { score: { $meta: 'textScore' } }
).sort({ score: { $meta: 'textScore' } });
```

### Indexing

```javascript
// Single field index
userSchema.index({ email: 1 });

// Compound index
userSchema.index({ username: 1, createdAt: -1 });

// Sparse index (only documents with the field)
userSchema.index({ 'profile.avatar': 1 }, { sparse: true });

// TTL index (automatic expiration)
userSchema.index({ createdAt: 1 }, { expireAfterSeconds: 3600 });

// Partial index
userSchema.index(
  { balance: 1 },
  { partialFilterExpression: { balance: { $gt: 0 } } }
);
```

### Connection Management

```javascript
// Multiple connections
const dbMain = mongoose.createConnection('mongodb://localhost/main');
const dbLogs = mongoose.createConnection('mongodb://localhost/logs');

const User = dbMain.model('User', userSchema);
const Log = dbLogs.model('Log', logSchema);

// Connection events
mongoose.connection.on('connected', () => {
  console.log('Connected to MongoDB');
});

mongoose.connection.on('error', (err) => {
  console.error('MongoDB connection error:', err);
});

mongoose.connection.on('disconnected', () => {
  console.log('Disconnected from MongoDB');
});

// Graceful shutdown
process.on('SIGINT', async () => {
  await mongoose.connection.close();
  process.exit(0);
});
```

## PostgreSQL with node-postgres

### Installation & Setup

```bash
npm install pg
npm install @types/pg  # For TypeScript
```

### Connection with Pool

```javascript
const { Pool } = require('pg');

// Connection pool setup
const pool = new Pool({
  user: 'username',
  password: 'password',
  host: 'localhost',
  port: 5432,
  database: 'trading_app',
  max: 20, // Maximum clients in pool
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Environment-based configuration
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false
});

// Pool events
pool.on('error', (err, client) => {
  console.error('Unexpected error on idle client', err);
  process.exit(-1);
});

pool.on('connect', (client) => {
  console.log('New client connected');
});
```

### Database Schema

```sql
-- Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(30) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    avatar_url TEXT,
    bio TEXT,
    balance DECIMAL(15,2) DEFAULT 0.00,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Trades table
CREATE TABLE trades (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    symbol VARCHAR(20) NOT NULL,
    quantity DECIMAL(15,8) NOT NULL,
    price DECIMAL(15,2) NOT NULL,
    trade_type VARCHAR(4) CHECK (trade_type IN ('buy', 'sell')),
    executed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    INDEX idx_trades_user_symbol (user_id, symbol),
    INDEX idx_trades_executed_at (executed_at)
);

-- Wallet addresses table
CREATE TABLE wallet_addresses (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    address VARCHAR(42) UNIQUE NOT NULL,
    chain VARCHAR(20) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_trades_symbol_date ON trades(symbol, executed_at);
CREATE INDEX idx_wallet_addresses_user ON wallet_addresses(user_id);

-- Updated timestamp trigger
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at 
    BEFORE UPDATE ON users 
    FOR EACH ROW 
    EXECUTE FUNCTION update_updated_at_column();
```

### Basic Queries

```javascript
// Simple query function
const query = (text, params) => pool.query(text, params);

// Create user
async function createUser(userData) {
  const { username, email, passwordHash, firstName, lastName } = userData;
  
  const result = await query(
    `INSERT INTO users (username, email, password_hash, first_name, last_name)
     VALUES ($1, $2, $3, $4, $5)
     RETURNING id, username, email, created_at`,
    [username, email, passwordHash, firstName, lastName]
  );
  
  return result.rows[0];
}

// Get user by ID
async function getUserById(userId) {
  const result = await query(
    'SELECT id, username, email, first_name, last_name, balance, settings FROM users WHERE id = $1',
    [userId]
  );
  
  return result.rows[0];
}

// Get user with trades
async function getUserWithTrades(userId) {
  const result = await query(`
    SELECT 
      u.id, u.username, u.email, u.balance,
      COALESCE(
        JSON_AGG(
          JSON_BUILD_OBJECT(
            'id', t.id,
            'symbol', t.symbol,
            'quantity', t.quantity,
            'price', t.price,
            'type', t.trade_type,
            'executed_at', t.executed_at
          )
        ) FILTER (WHERE t.id IS NOT NULL),
        '[]'
      ) as trades
    FROM users u
    LEFT JOIN trades t ON u.id = t.user_id
    WHERE u.id = $1
    GROUP BY u.id
  `, [userId]);
  
  return result.rows[0];
}

// Update user balance
async function updateUserBalance(userId, amount) {
  const result = await query(
    `UPDATE users 
     SET balance = balance + $2, updated_at = NOW()
     WHERE id = $1
     RETURNING balance`,
    [userId, amount]
  );
  
  return result.rows[0]?.balance;
}
```

### Transactions

```javascript
async function transferFunds(fromUserId, toUserId, amount) {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // Check sender balance
    const senderResult = await client.query(
      'SELECT balance FROM users WHERE id = $1 FOR UPDATE',
      [fromUserId]
    );
    
    if (!senderResult.rows[0] || senderResult.rows[0].balance < amount) {
      throw new Error('Insufficient balance');
    }
    
    // Debit sender
    await client.query(
      'UPDATE users SET balance = balance - $2 WHERE id = $1',
      [fromUserId, amount]
    );
    
    // Credit receiver
    await client.query(
      'UPDATE users SET balance = balance + $2 WHERE id = $1',
      [toUserId, amount]
    );
    
    // Log transaction
    await client.query(
      `INSERT INTO transactions (from_user_id, to_user_id, amount, type)
       VALUES ($1, $2, $3, 'transfer')`,
      [fromUserId, toUserId, amount]
    );
    
    await client.query('COMMIT');
    return true;
    
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

### Advanced Queries

```javascript
// Complex aggregation
async function getTradingStats(userId, timeframe = '30 days') {
  const result = await query(`
    WITH trade_stats AS (
      SELECT
        symbol,
        COUNT(*) as trade_count,
        SUM(CASE WHEN trade_type = 'buy' THEN quantity ELSE 0 END) as total_bought,
        SUM(CASE WHEN trade_type = 'sell' THEN quantity ELSE 0 END) as total_sold,
        AVG(price) as avg_price,
        SUM(quantity * price) as total_volume
      FROM trades
      WHERE user_id = $1 
        AND executed_at >= NOW() - INTERVAL $2
      GROUP BY symbol
    )
    SELECT 
      *,
      total_bought - total_sold as net_position
    FROM trade_stats
    ORDER BY total_volume DESC
  `, [userId, timeframe]);
  
  return result.rows;
}

// Pagination with cursor
async function getTradeHistory(userId, limit = 20, cursor = null) {
  let whereClause = 'WHERE user_id = $1';
  let params = [userId];
  
  if (cursor) {
    whereClause += ' AND executed_at < $2';
    params.push(cursor);
  }
  
  const result = await query(`
    SELECT id, symbol, quantity, price, trade_type, executed_at
    FROM trades
    ${whereClause}
    ORDER BY executed_at DESC
    LIMIT $${params.length + 1}
  `, [...params, limit]);
  
  const trades = result.rows;
  const hasNextPage = trades.length === limit;
  const nextCursor = hasNextPage ? trades[trades.length - 1].executed_at : null;
  
  return {
    trades: hasNextPage ? trades.slice(0, -1) : trades,
    hasNextPage,
    nextCursor
  };
}
```

### Connection Management

```javascript
// Database adapter pattern
class Database {
  constructor(pool) {
    this.pool = pool;
  }
  
  async query(text, params) {
    const start = Date.now();
    const res = await this.pool.query(text, params);
    const duration = Date.now() - start;
    
    console.log('Executed query', { text, duration, rows: res.rowCount });
    return res;
  }
  
  async getClient() {
    const client = await this.pool.connect();
    const query = client.query;
    const release = client.release;
    
    // Set timeout for client leaks
    const timeout = setTimeout(() => {
      console.error('A client has been checked out for more than 5 seconds!');
    }, 5000);
    
    // Monkey patch for tracking
    client.query = (...args) => {
      client.lastQuery = args;
      return query.apply(client, args);
    };
    
    client.release = () => {
      clearTimeout(timeout);
      client.query = query;
      client.release = release;
      return release.apply(client);
    };
    
    return client;
  }
  
  async end() {
    await this.pool.end();
  }
}

const db = new Database(pool);
```

## Database Design Patterns

### Repository Pattern

```javascript
// User Repository
class UserRepository {
  constructor(db) {
    this.db = db;
  }
  
  async create(userData) {
    // MongoDB
    const user = new User(userData);
    return await user.save();
    
    // PostgreSQL
    // return await this.db.query('INSERT INTO users...', [userData]);
  }
  
  async findById(id) {
    // MongoDB
    return await User.findById(id);
    
    // PostgreSQL
    // const result = await this.db.query('SELECT * FROM users WHERE id = $1', [id]);
    // return result.rows[0];
  }
  
  async findByEmail(email) {
    // MongoDB
    return await User.findOne({ email });
    
    // PostgreSQL
    // const result = await this.db.query('SELECT * FROM users WHERE email = $1', [email]);
    // return result.rows[0];
  }
  
  async update(id, updates) {
    // MongoDB
    return await User.findByIdAndUpdate(id, updates, { new: true });
    
    // PostgreSQL
    // return await this.db.query('UPDATE users SET ... WHERE id = $1', [id]);
  }
  
  async delete(id) {
    // MongoDB
    return await User.findByIdAndDelete(id);
    
    // PostgreSQL
    // return await this.db.query('DELETE FROM users WHERE id = $1', [id]);
  }
}
```

### Data Access Layer

```javascript
class DataAccessLayer {
  constructor(config) {
    this.dbType = config.type; // 'mongodb' or 'postgresql'
    this.connection = this.initializeConnection(config);
  }
  
  initializeConnection(config) {
    if (config.type === 'mongodb') {
      return mongoose.createConnection(config.url);
    } else {
      return new Pool(config);
    }
  }
  
  async executeQuery(operation, params) {
    if (this.dbType === 'mongodb') {
      return await this.executeMongoOperation(operation, params);
    } else {
      return await this.executePostgresQuery(operation, params);
    }
  }
  
  async executeMongoOperation(operation, params) {
    // Implement MongoDB operations
  }
  
  async executePostgresQuery(query, params) {
    // Implement PostgreSQL operations
  }
}
```

## Performance Optimization

### MongoDB Optimization

```javascript
// Efficient queries
// Use projection to limit fields
const users = await User.find({}, 'username email balance');

// Use lean() for read-only operations
const users = await User.find({}).lean();

// Proper indexing
userSchema.index({ email: 1 });
userSchema.index({ username: 1, createdAt: -1 });

// Aggregation optimization
const stats = await User.aggregate([
  { $match: { balance: { $gt: 0 } } }, // Filter early
  { $group: { _id: null, avgBalance: { $avg: '$balance' } } }
]);

// Connection pooling
mongoose.connect(uri, {
  maxPoolSize: 10,
  bufferMaxEntries: 0
});
```

### PostgreSQL Optimization

```javascript
// Use prepared statements for repeated queries
const preparedQuery = {
  name: 'get-user-by-id',
  text: 'SELECT * FROM users WHERE id = $1',
  values: [userId]
};

// Connection pooling optimization
const pool = new Pool({
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
  maxUses: 7500 // Rotate connections
});

// Batch operations
async function batchInsertTrades(trades) {
  const values = trades.map(trade => 
    `(${trade.userId}, '${trade.symbol}', ${trade.quantity}, ${trade.price}, '${trade.type}')`
  ).join(',');
  
  return await query(`
    INSERT INTO trades (user_id, symbol, quantity, price, trade_type)
    VALUES ${values}
  `);
}
```

## Error Handling

### MongoDB Error Handling

```javascript
try {
  await User.create(userData);
} catch (error) {
  if (error.code === 11000) {
    // Duplicate key error
    throw new Error('User already exists');
  } else if (error.name === 'ValidationError') {
    // Validation error
    const messages = Object.values(error.errors).map(err => err.message);
    throw new Error(`Validation failed: ${messages.join(', ')}`);
  } else {
    throw error;
  }
}
```

### PostgreSQL Error Handling

```javascript
try {
  await query('INSERT INTO users...', params);
} catch (error) {
  if (error.code === '23505') {
    // Unique violation
    throw new Error('User already exists');
  } else if (error.code === '23503') {
    // Foreign key violation
    throw new Error('Referenced record does not exist');
  } else if (error.code === '23514') {
    // Check constraint violation
    throw new Error('Data validation failed');
  } else {
    throw error;
  }
}
```

## Security Best Practices

### Input Validation & Sanitization

```javascript
// MongoDB - Use Mongoose validation
const userSchema = new Schema({
  email: {
    type: String,
    required: true,
    validate: {
      validator: function(v) {
        return /^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$/.test(v);
      },
      message: 'Invalid email format'
    }
  }
});

// PostgreSQL - Parameterized queries prevent SQL injection
const result = await query(
  'SELECT * FROM users WHERE email = $1 AND active = $2',
  [email, true]
);
```

### Password Security

```javascript
const bcrypt = require('bcrypt');

// Hash password before storing
async function hashPassword(password) {
  const saltRounds = 12;
  return await bcrypt.hash(password, saltRounds);
}

// Verify password
async function verifyPassword(password, hash) {
  return await bcrypt.compare(password, hash);
}
```

### Environment Configuration

```javascript
// Use environment variables for sensitive data
const config = {
  mongodb: {
    uri: process.env.MONGODB_URI,
    options: {
      useNewUrlParser: true,
      useUnifiedTopology: true
    }
  },
  postgresql: {
    connectionString: process.env.DATABASE_URL,
    ssl: process.env.NODE_ENV === 'production'
  }
};
```

## Migration Strategies

### MongoDB Migrations

```javascript
// MongoDB migration script
async function migrateUsers() {
  const users = await User.find({ version: { $ne: 2 } });
  
  for (const user of users) {
    // Transform data
    user.settings = {
      notifications: user.notifications || true,
      theme: user.theme || 'light'
    };
    user.version = 2;
    
    await user.save();
  }
}
```

### PostgreSQL Migrations

```sql
-- Migration: Add settings column
ALTER TABLE users ADD COLUMN settings JSONB DEFAULT '{}';

-- Migration: Update existing data
UPDATE users 
SET settings = jsonb_build_object(
  'notifications', COALESCE(notifications, true),
  'theme', COALESCE(theme, 'light')
)
WHERE settings = '{}';

-- Migration: Remove old columns
ALTER TABLE users DROP COLUMN notifications;
ALTER TABLE users DROP COLUMN theme;
```

## Monitoring & Maintenance

### Performance Monitoring

```javascript
// MongoDB monitoring
mongoose.connection.on('connected', () => {
  console.log('MongoDB connected');
});

mongoose.set('debug', process.env.NODE_ENV === 'development');

// PostgreSQL monitoring
pool.on('acquire', () => {
  console.log('Client acquired from pool');
});

pool.on('error', (err) => {
  console.error('Pool error:', err);
});

// Query performance logging
const originalQuery = pool.query;
pool.query = function(text, params, callback) {
  const start = Date.now();
  return originalQuery.call(this, text, params, (err, result) => {
    const duration = Date.now() - start;
    console.log('Query executed', { text, duration, rows: result?.rowCount });
    if (callback) callback(err, result);
  });
};
```

### Health Checks

```javascript
// Database health check endpoint
app.get('/health/db', async (req, res) => {
  try {
    // MongoDB health
    if (mongoose.connection.readyState === 1) {
      await mongoose.connection.db.admin().ping();
    }
    
    // PostgreSQL health
    await pool.query('SELECT 1');
    
    res.json({ status: 'healthy', timestamp: new Date() });
  } catch (error) {
    res.status(503).json({ 
      status: 'unhealthy', 
      error: error.message,
      timestamp: new Date()
    });
  }
});
```

## Best Practices Summary

1. **Connection Management**: Use connection pooling and handle connection events
2. **Error Handling**: Implement comprehensive error handling with specific error codes
3. **Security**: Use parameterized queries, validate inputs, hash passwords
4. **Performance**: Implement proper indexing, use projections, optimize queries
5. **Monitoring**: Log query performance, implement health checks
6. **Schema Design**: Design schemas for your specific use case and query patterns
7. **Transactions**: Use transactions for data consistency in multi-step operations
8. **Environment**: Use environment variables for configuration
9. **Testing**: Write comprehensive tests for database operations
10. **Documentation**: Document your schema and API for team collaboration

## Resources

- [Mongoose Documentation](https://mongoosejs.com/)
- [MongoDB Manual](https://docs.mongodb.com/)
- [node-postgres Documentation](https://node-postgres.com/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Database Design Patterns](https://www.patterns.dev/posts/data-patterns) 