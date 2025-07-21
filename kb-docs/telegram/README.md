# Telegram Bot API Documentation

## Overview

This documentation covers building Telegram bots using two popular Node.js libraries: `node-telegram-bot-api` and `grammY`. Both provide powerful interfaces for creating interactive Telegram bots with different approaches to handling updates and managing bot state.

## Table of Contents

1. [Getting Started](#getting-started)
2. [node-telegram-bot-api](#node-telegram-bot-api)
3. [grammY Framework](#grammy-framework)
4. [Bot Setup & Configuration](#bot-setup--configuration)
5. [Message Handling](#message-handling)
6. [Commands](#commands)
7. [Inline Keyboards & Callbacks](#inline-keyboards--callbacks)
8. [File Handling](#file-handling)
9. [WebHooks vs Polling](#webhooks-vs-polling)
10. [Error Handling](#error-handling)
11. [Best Practices](#best-practices)

## Getting Started

### Creating a Bot with BotFather

1. Message [@BotFather](https://t.me/BotFather) on Telegram
2. Send `/newbot` and follow the instructions
3. Choose a name and username for your bot
4. Save the API token provided by BotFather

### Environment Setup

```bash
# Create project directory
mkdir telegram-bot
cd telegram-bot
npm init -y

# Install dependencies (choose one)
npm install node-telegram-bot-api
# OR
npm install grammy
```

## node-telegram-bot-api

### Installation & Basic Setup

```bash
npm install node-telegram-bot-api
npm install --save-dev @types/node-telegram-bot-api  # For TypeScript
```

### Basic Bot Example

```javascript
const TelegramBot = require('node-telegram-bot-api');

const token = 'YOUR_TELEGRAM_BOT_TOKEN';
const bot = new TelegramBot(token, { polling: true });

// Echo command
bot.onText(/\/echo (.+)/, (msg, match) => {
  const chatId = msg.chat.id;
  const resp = match[1];
  bot.sendMessage(chatId, resp);
});

// Listen for any message
bot.on('message', (msg) => {
  const chatId = msg.chat.id;
  bot.sendMessage(chatId, 'Received your message');
});
```

### File Handling with node-telegram-bot-api

```javascript
// Send photo from file path
bot.sendPhoto(chatId, 'path/to/photo.jpg');

// Send photo from URL
bot.sendPhoto(chatId, 'https://example.com/photo.jpg');

// Send photo from Buffer
const buffer = fs.readFileSync('path/to/photo.jpg');
bot.sendPhoto(chatId, buffer);

// Send audio with metadata
const fileOptions = {
  filename: 'song.mp3',
  contentType: 'audio/mpeg',
};
bot.sendAudio(chatId, audioStream, {}, fileOptions);

// Send document
bot.sendDocument(chatId, 'path/to/document.pdf');
```

### Error Handling

```javascript
// Handle polling errors
bot.on('polling_error', (error) => {
  console.log('Polling error:', error.code);
});

// Handle webhook errors
bot.on('webhook_error', (error) => {
  console.log('Webhook error:', error.code);
});

// Handle API errors
bot.sendMessage(chatId, 'text').catch((error) => {
  console.log('API error:', error.code);
  console.log('Response:', error.response.body);
});
```

## grammY Framework

### Installation & Basic Setup

```bash
npm install grammy
```

### Basic Bot Example

```typescript
import { Bot } from 'grammy';

const token = 'YOUR_TELEGRAM_BOT_TOKEN';
const bot = new Bot(token);

// Command handlers
bot.command('start', (ctx) => ctx.reply('Welcome!'));
bot.command('help', (ctx) => ctx.reply('Help information'));

// Text message handler
bot.on('message:text', (ctx) => ctx.reply('Got your text message!'));

// Start the bot
bot.start();
```

### Context Object in grammY

```typescript
bot.on('message', async (ctx) => {
  // Access message info
  const messageText = ctx.message?.text;
  const userId = ctx.from?.id;
  const chatId = ctx.chat.id;
  
  // Reply to the message
  await ctx.reply('Response text');
  
  // Reply with quote
  await ctx.reply('Quoted response', {
    reply_parameters: { message_id: ctx.message.message_id }
  });
});
```

### Advanced grammY Features

#### Session Management
```typescript
import { Bot, session, SessionFlavor } from 'grammy';

interface SessionData {
  pizzaCount: number;
}

type MyContext = Context & SessionFlavor<SessionData>;

const bot = new Bot<MyContext>(token);

// Session middleware
function initial(): SessionData {
  return { pizzaCount: 0 };
}
bot.use(session({ initial }));

bot.command('hunger', async (ctx) => {
  const count = ctx.session.pizzaCount;
  await ctx.reply(`Your hunger level is ${count}!`);
});

bot.hears(/.*🍕.*/, (ctx) => ctx.session.pizzaCount++);
```

#### Router for Complex Logic
```typescript
import { Router } from 'grammy';

const router = new Router((ctx) => {
  // Determine route based on user state
  return ctx.session?.step || 'start';
});

router.route('start', async (ctx) => {
  ctx.session.step = 'askName';
  await ctx.reply('What is your name?');
});

router.route('askName', async (ctx) => {
  ctx.session.name = ctx.message?.text;
  ctx.session.step = 'complete';
  await ctx.reply(`Hello, ${ctx.session.name}!`);
});

bot.use(router);
```

#### Custom Context Types
```typescript
interface BotConfig {
  botDeveloper: number;
  isDeveloper: boolean;
}

type MyContext = Context & {
  config: BotConfig;
};

const bot = new Bot<MyContext>(token);

// Middleware to set custom properties
bot.use(async (ctx, next) => {
  ctx.config = {
    botDeveloper: 123456,
    isDeveloper: ctx.from?.id === 123456,
  };
  await next();
});

bot.command('start', async (ctx) => {
  if (ctx.config.isDeveloper) {
    await ctx.reply('Hi admin!');
  } else {
    await ctx.reply('Welcome user!');
  }
});
```

## Bot Setup & Configuration

### Polling vs Webhooks

#### Polling (Development)
```javascript
// node-telegram-bot-api
const bot = new TelegramBot(token, { polling: true });

// grammY
const bot = new Bot(token);
bot.start(); // Uses long polling by default
```

#### Webhooks (Production)
```javascript
// node-telegram-bot-api
bot.setWebHook('https://your-domain.com/webhook', {
  certificate: 'path/to/cert.pem'
});

// grammY with Express
import express from 'express';
import { webhookCallback } from 'grammy';

const app = express();
app.use(express.json());
app.use(webhookCallback(bot, 'express'));
app.listen(3000);
```

### SSL Certificate Setup
```bash
# Generate private key
openssl genrsa -out key.pem 2048

# Generate certificate
openssl req -new -sha256 -key key.pem -out cert.pem
```

## Message Handling

### Different Message Types

```typescript
// grammY examples
bot.on('message:text', (ctx) => { /* handle text */ });
bot.on('message:photo', (ctx) => { /* handle photos */ });
bot.on('message:audio', (ctx) => { /* handle audio */ });
bot.on('message:video', (ctx) => { /* handle video */ });
bot.on('message:document', (ctx) => { /* handle documents */ });
bot.on('message:sticker', (ctx) => { /* handle stickers */ });

// Text pattern matching
bot.hears(/hello/i, (ctx) => ctx.reply('Hello there!'));
bot.hears('exact match', (ctx) => ctx.reply('Exact match found!'));
```

### Filter Queries in grammY
```typescript
// Combine filters
bot.on(':text').command('help'); // Text messages with /help command
bot.chatType('private').command('start'); // Private chat commands only

// Custom filters
bot.on('message', (ctx, next) => {
  if (ctx.message?.text?.includes('urgent')) {
    return next(); // Continue to next handler
  }
  // Skip if not urgent
});
```

## Commands

### Command Registration

```typescript
// Basic commands
bot.command('start', (ctx) => ctx.reply('Bot started!'));
bot.command('help', (ctx) => ctx.reply('Available commands...'));

// Multiple commands
bot.command(['info', 'about'], (ctx) => ctx.reply('Bot information'));

// Commands with parameters
bot.command('weather', async (ctx) => {
  const city = ctx.match; // Text after /weather
  if (!city) {
    return ctx.reply('Please specify a city: /weather London');
  }
  await ctx.reply(`Weather for ${city}: ...`);
});
```

### Command Groups (grammY)
```typescript
import { CommandGroup } from '@grammyjs/commands';

const myCommands = new CommandGroup();

myCommands.command('hello', 'Say hello', (ctx) => 
  ctx.reply('Hello, world!')
);

myCommands.command('start', 'Start the bot', (ctx) => 
  ctx.reply('Starting...')
);

bot.use(myCommands);

// Set commands in Telegram menu
await myCommands.setCommands(bot);
```

## Inline Keyboards & Callbacks

### Creating Inline Keyboards

```javascript
// node-telegram-bot-api
const keyboard = {
  inline_keyboard: [
    [
      { text: 'Button 1', callback_data: 'btn1' },
      { text: 'Button 2', callback_data: 'btn2' }
    ],
    [
      { text: 'URL Button', url: 'https://example.com' }
    ]
  ]
};

bot.sendMessage(chatId, 'Choose an option:', {
  reply_markup: keyboard
});

// Handle callbacks
bot.on('callback_query', (callbackQuery) => {
  const action = callbackQuery.data;
  const msg = callbackQuery.message;
  
  if (action === 'btn1') {
    bot.answerCallbackQuery(callbackQuery.id, 'Button 1 pressed!');
  }
});
```

```typescript
// grammY
import { InlineKeyboard } from 'grammy';

const keyboard = new InlineKeyboard()
  .text('Button 1', 'btn1')
  .text('Button 2', 'btn2')
  .row()
  .url('Visit Website', 'https://example.com');

await ctx.reply('Choose an option:', {
  reply_markup: keyboard
});

// Handle specific callbacks
bot.callbackQuery('btn1', async (ctx) => {
  await ctx.answerCallbackQuery('Button 1 pressed!');
  await ctx.editMessageText('You pressed Button 1');
});

// Handle all callbacks
bot.on('callback_query:data', async (ctx) => {
  const data = ctx.callbackQuery.data;
  await ctx.answerCallbackQuery(`You pressed: ${data}`);
});
```

### Regular Keyboards

```typescript
// Custom keyboard
const keyboard = {
  keyboard: [
    ['📊 Stats', '💰 Balance'],
    ['📈 Markets', '⚙️ Settings']
  ],
  resize_keyboard: true,
  one_time_keyboard: true
};

await bot.sendMessage(chatId, 'Main menu:', {
  reply_markup: keyboard
});

// Remove keyboard
await bot.sendMessage(chatId, 'Keyboard removed', {
  reply_markup: { remove_keyboard: true }
});
```

## File Handling

### Sending Files

```typescript
// grammY file handling
import { InputFile } from 'grammy';

// From file path
await ctx.replyWithPhoto(new InputFile('path/to/photo.jpg'));

// From URL
await ctx.replyWithPhoto('https://example.com/photo.jpg');

// From Buffer
const buffer = fs.readFileSync('photo.jpg');
await ctx.replyWithPhoto(new InputFile(buffer, 'photo.jpg'));

// With caption
await ctx.replyWithPhoto(new InputFile('photo.jpg'), {
  caption: 'Photo description'
});
```

### Receiving Files

```typescript
// Handle photo uploads
bot.on('message:photo', async (ctx) => {
  const photo = ctx.message.photo;
  const fileId = photo[photo.length - 1].file_id; // Largest size
  
  // Get file info
  const file = await ctx.api.getFile(fileId);
  const filePath = file.file_path;
  
  // Download file
  const fileUrl = `https://api.telegram.org/file/bot${token}/${filePath}`;
  // Use fetch or axios to download
});
```

## WebHooks vs Polling

### When to Use Each

**Polling (Development)**:
- Easy to set up
- Good for development/testing
- No SSL certificate required
- Bot can be behind NAT/firewall

**Webhooks (Production)**:
- More efficient for high-volume bots
- Real-time updates
- Better for scaling
- Requires public HTTPS endpoint

### Webhook Setup with Express

```typescript
import express from 'express';
import { webhookCallback } from 'grammy';

const app = express();
app.use(express.json());

// grammY webhook
app.use('/webhook', webhookCallback(bot, 'express'));

// Start server
app.listen(3000, () => {
  console.log('Webhook server running on port 3000');
});

// Set webhook
await bot.api.setWebhook('https://your-domain.com/webhook');
```

## Error Handling

### Comprehensive Error Handling

```typescript
// grammY error handling
bot.catch((err) => {
  const ctx = err.ctx;
  console.error(`Error while handling update ${ctx.update.update_id}:`);
  const e = err.error;
  
  if (e instanceof GrammyError) {
    console.error('Error in request:', e.description);
  } else if (e instanceof HttpError) {
    console.error('Could not contact Telegram:', e);
  } else {
    console.error('Unknown error:', e);
  }
});

// Graceful error handling in handlers
bot.command('test', async (ctx) => {
  try {
    // Potentially failing operation
    await someRiskyOperation();
    await ctx.reply('Success!');
  } catch (error) {
    console.error('Operation failed:', error);
    await ctx.reply('Sorry, something went wrong!');
  }
});
```

## Best Practices

### 1. Security

```typescript
// Validate user input
bot.on('message:text', async (ctx) => {
  const text = ctx.message.text;
  
  // Sanitize input
  if (text.length > 1000) {
    return ctx.reply('Message too long!');
  }
  
  // Validate format
  if (!/^[a-zA-Z0-9\s]+$/.test(text)) {
    return ctx.reply('Invalid characters in message!');
  }
});

// Rate limiting
const userLastMessage = new Map();

bot.use(async (ctx, next) => {
  const userId = ctx.from?.id;
  const now = Date.now();
  const lastMessage = userLastMessage.get(userId);
  
  if (lastMessage && now - lastMessage < 1000) {
    return; // Rate limit: 1 message per second
  }
  
  userLastMessage.set(userId, now);
  await next();
});
```

### 2. State Management

```typescript
// Use sessions for user state
interface UserSession {
  step: string;
  data: any;
}

// Persistent sessions with database
bot.use(session({
  initial: (): UserSession => ({ step: 'start', data: {} }),
  storage: new DatabaseAdapter(), // Custom storage adapter
}));
```

### 3. Environment Configuration

```typescript
// Use environment variables
const config = {
  botToken: process.env.BOT_TOKEN!,
  webhookUrl: process.env.WEBHOOK_URL,
  port: parseInt(process.env.PORT || '3000'),
  dbUrl: process.env.DATABASE_URL!,
};

// Validate required config
if (!config.botToken) {
  throw new Error('BOT_TOKEN environment variable is required');
}
```

### 4. Logging

```typescript
// Structured logging
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'bot.log' }),
    new winston.transports.Console()
  ]
});

bot.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  const duration = Date.now() - start;
  
  logger.info('Update processed', {
    updateId: ctx.update.update_id,
    userId: ctx.from?.id,
    duration,
  });
});
```

### 5. Graceful Shutdown

```typescript
// Handle shutdown signals
process.once('SIGINT', () => bot.stop());
process.once('SIGTERM', () => bot.stop());

// Cleanup on exit
bot.on('stop', () => {
  console.log('Bot stopped gracefully');
  // Close database connections, etc.
});
```

## Useful Libraries

- **@grammyjs/conversations**: For complex conversation flows
- **@grammyjs/menu**: For dynamic menus
- **@grammyjs/i18n**: For internationalization
- **@grammyjs/runner**: For concurrent update processing
- **@grammyjs/auto-retry**: For automatic retry on API errors
- **@grammyjs/transformer-throttler**: For API rate limiting

## Resources

- [Telegram Bot API Documentation](https://core.telegram.org/bots/api)
- [node-telegram-bot-api GitHub](https://github.com/yagop/node-telegram-bot-api)
- [grammY Documentation](https://grammy.dev/)
- [BotFather](https://t.me/BotFather) - Create and manage bots
- [Telegram Bot Examples](https://core.telegram.org/bots/samples) 