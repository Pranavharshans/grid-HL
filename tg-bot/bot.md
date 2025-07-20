# Hyperliquid Telegram Grid Trading Bot - System Architecture & Design

## 🎯 Project Overview
This directory contains the design and implementation for a Telegram bot that enables users to create, manage, and monitor grid trading strategies on Hyperliquid DEX directly through Telegram commands.

## 🏗️ System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TELEGRAM CLIENT LAYER                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │   Mobile    │  │   Desktop   │  │  Web App    │  │   Bot API   │ │
│  │   Client    │  │   Client    │  │   Client    │  │   Testing   │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                           ┌───────▼───────┐
                           │  TELEGRAM BOT │
                           │     API       │
                           └───────┬───────┘
                                   │
┌─────────────────────────────────▼─────────────────────────────────┐
│                        BOT SERVER LAYER                           │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                   COMMAND ROUTER                           │  │
│  │  /start  /wallet  /grid  /status  /help  /settings       │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                   │                                │
│  ┌─────────────────────────────────▼─────────────────────────────┐  │
│  │                  AUTHENTICATION & SESSION                  │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │  │
│  │  │  User Auth  │  │ Wallet Auth │  │   Session   │        │  │
│  │  │  Manager    │  │  Manager    │  │   Manager   │        │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘        │  │
│  └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                   │
┌─────────────────────────────────▼─────────────────────────────────┐
│                     BUSINESS LOGIC LAYER                          │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │
│  │  Grid Trading │  │  Portfolio    │  │   Risk        │        │
│  │     Engine    │  │   Manager     │  │  Management   │        │
│  │               │  │               │  │               │        │
│  │ • Strategy    │  │ • Balance     │  │ • Stop Loss   │        │
│  │ • Execution   │  │ • P&L Track   │  │ • Position    │        │
│  │ • Monitoring  │  │ • Analytics   │  │ • Limits      │        │
│  └───────────────┘  └───────────────┘  └───────────────┘        │
│                                   │                              │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │
│  │  Wallet       │  │   XP/Rewards  │  │   Referral    │        │
│  │  Connection   │  │    System     │  │    System     │        │
│  │               │  │               │  │               │        │
│  │ • QR Code +   │  │ • Volume XP   │  │ • Invite Codes│        │
│  │   Buttons     │  │ • Leaderboard │  │ • Commission  │        │
│  │ • Session Mgmt│  │ • Achievements│  │ • Bonuses     │        │
│  └───────────────┘  └───────────────┘  └───────────────┘        │
└─────────────────────────────────────────────────────────────────┘
                                   │
┌─────────────────────────────────▼─────────────────────────────────┐
│                      INTEGRATION LAYER                            │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                 HYPERLIQUID ADAPTER                         │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │  │
│  │  │   Exchange  │  │    Info     │  │  WebSocket  │        │  │
│  │  │   Client    │  │   Client    │  │   Manager   │        │  │
│  │  │             │  │             │  │             │        │  │
│  │  │ • Orders    │  │ • Prices    │  │ • Real-time │        │  │
│  │  │ • Cancel    │  │ • Balances  │  │ • Fills     │        │  │
│  │  │ • Modify    │  │ • Positions │  │ • Events    │        │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘        │  │
│  └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                   │
┌─────────────────────────────────▼─────────────────────────────────┐
│                        DATA LAYER                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │    PostgreSQL   │  │      Redis      │  │   File Storage  │    │
│  │   (Primary DB)  │  │     (Cache)     │  │    (Logs/Keys)  │    │
│  │                 │  │                 │  │                 │    │
│  │ • Users         │  │ • Sessions      │  │ • Encrypted     │    │
│  │ • Wallets       │  │ • Price Cache   │  │   Private Keys  │    │
│  │ • Grid Configs  │  │ • Order Cache   │  │ • Trade Logs    │    │
│  │ • Transactions  │  │ • Rate Limits   │  │ • Error Logs    │    │
│  │ • Referrals     │  │ • Temp Data     │  │ • Backups       │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                                   │
┌─────────────────────────────────▼─────────────────────────────────┐
│                    EXTERNAL SERVICES                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │   Hyperliquid   │  │   Price Feeds   │  │   Monitoring    │    │
│  │      DEX        │  │   (Backup)      │  │   Services      │    │
│  │                 │  │                 │  │                 │    │
│  │ • REST API      │  │ • CoinGecko     │  │ • Sentry        │    │
│  │ • WebSocket     │  │ • Binance       │  │ • DataDog       │    │
│  │ • Blockchain    │  │ • Coinbase      │  │ • Prometheus    │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

## 📋 Module Structure & Flow

### 1. Command Processing Flow
```
User Input → Command Parser → Authentication Check → Handler Execution
     │              │                 │                    │
     ▼              ▼                 ▼                    ▼
/grid create    Route to         Check user        Execute grid
parameters   → GridHandler →   authentication → creation logic
```

### 2. Grid Trading Engine Flow
```
Grid Creation → Parameter Validation → Order Placement → Monitoring Loop
      │                  │                   │                │
      ▼                  ▼                   ▼                ▼
  User inputs →    Validate range    →   Place initial   →  Check fills
  via Telegram     price, amount        orders on HL      & rebalance
```

### 3. WebSocket Data Flow
```
Hyperliquid WS → Message Router → Strategy Engine → Notification System
      │               │                 │                   │
      ▼               ▼                 ▼                   ▼
  Price/Fill    →  Parse & filter →  Update grid    →  Notify user
   updates         by user/grid       positions         via Telegram
```

## 🔧 Core Modules Design

### A. Command Handlers Module
```
/src/handlers/
├── index.ts           # Main handler router
├── start.handler.ts   # Welcome & onboarding
├── wallet.handler.ts  # Wallet connection & management
├── grid.handler.ts    # Grid creation & management
├── status.handler.ts  # Portfolio & position status
├── settings.handler.ts# User preferences
└── help.handler.ts    # Documentation & support
```

### B. Grid Trading Module
```
/src/trading/
├── GridEngine.ts      # Core grid strategy logic
├── OrderManager.ts    # Order placement & tracking
├── RiskManager.ts     # Risk controls & limits
├── PnLCalculator.ts   # Profit/loss calculations
└── GridValidator.ts   # Parameter validation
```

### C. Wallet Connection Module
```
/src/wallet/
├── WalletHandler.py      # Main wallet connection logic
├── WalletConnect.py      # WalletConnect protocol integration
├── QRGenerator.py        # QR code generation
├── SessionManager.py     # User session management
└── DeepLinks.py         # Mobile wallet deep link handling
```

### D. Hyperliquid Integration Module
```
/src/hyperliquid/
├── HLClient.py        # Main client wrapper (existing GridTrading)
├── ExchangeService.py # Trading operations
├── InfoService.py     # Market data & account info
├── WebSocketService.py# Real-time data streams
└── GridEngine.py      # Grid strategy engine (existing code)
```

### E. Database Models
```
/src/models/
├── User.py           # User accounts & preferences
├── WalletSession.py  # Active wallet sessions (in-memory)
├── GridConfig.py     # Grid trading configurations
├── Position.py       # Active positions & orders
├── Transaction.py    # Trade history & logs
└── Referral.py       # Referral system data
```

## 🎮 User Interaction Flow

### 1. Wallet Connection Process
```
/wallet → QR Code + Wallet Buttons → User Choice → Connection → Ready to Trade
   │              │                       │           │            │
   ▼              ▼                       ▼           ▼            ▼
Start flow → Show QR + clickable → Scan/Click → WalletConnect → Session active
            wallet options        wallet btn    handshake
```

### 2. Grid Creation Wizard
```
/grid create → Asset Selection → Price Range → Grid Settings → Confirmation
      │              │              │              │             │
      ▼              ▼              ▼              ▼             ▼
   Choose coin → Set min/max → Grid count/size → Review params → Deploy
```

### 3. Monitoring & Management
```
/status → Grid Overview → Select Grid → Actions Menu → Execute Action
    │           │             │            │             │
    ▼           ▼             ▼            ▼             ▼
View all → Pick specific → Modify/Stop → Confirm → Update strategy
```

## 🔐 Wallet Connection & Security

### 1. Universal Connection Methods
```
┌─────────────────────────────────────────────────────────┐
│                  WALLET CONNECTION                      │
│                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│  │  QR Code +  │    │ WalletConnect│    │   Session   │ │
│  │   Buttons   │ -> │  Protocol   │ -> │ Management  │ │
│  │             │    │             │    │             │ │
│  └─────────────┘    └─────────────┘    └─────────────┘ │
│         │                   │                   │      │
│         ▼                   ▼                   ▼      │
│  • Scan QR code     • Encrypted bridge   • In-memory   │
│  • Click wallet     • No private keys    • Session-based│
│  • Universal links  • User approval      • Auto-expire │
└─────────────────────────────────────────────────────────┘
```

### 2. Connection Flow Options
```
┌─────────────────────────────────────────────────────────┐
│                 CONNECTION METHODS                      │
│                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│  │   DESKTOP   │    │   MOBILE    │    │  FALLBACK   │ │
│  │             │    │             │    │             │ │
│  │ • Scan QR   │    │ • Tap wallet│    │ • Private   │ │
│  │   with      │    │   button    │    │   key input │ │
│  │   mobile    │    │ • Deep links│    │ • Always    │ │
│  │   wallet    │    │ • Auto-open │    │   works     │ │
│  └─────────────┘    └─────────────┘    └─────────────┘ │
└─────────────────────────────────────────────────────────┘
```

## 📊 Monitoring & Analytics

### 1. Real-time Metrics Dashboard
```
┌─────────────────────────────────────────────────────────┐
│                  MONITORING SYSTEM                      │
│                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│  │   System    │    │    User     │    │  Strategy   │ │
│  │  Metrics    │    │  Metrics    │    │  Metrics    │ │
│  │             │    │             │    │             │ │
│  │ • API calls │    │ • Active    │    │ • Grid      │ │
│  │ • Latency   │    │   users     │    │   profit    │ │
│  │ • Errors    │    │ • Volume    │    │ • Fill rate │ │
│  │ • Uptime    │    │ • Grids     │    │ • Risk      │ │
│  └─────────────┘    └─────────────┘    └─────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### 2. Alert System
```
High Priority: Position liquidation risk, API failures
Medium Priority: Grid completion, profit targets hit
Low Priority: Daily summaries, system updates
```

## 🚀 Implementation Roadmap

### Phase 1: Core Infrastructure (Week 1-2)
- [x] Set up TypeScript project structure
- [x] Implement Telegram bot framework
- [x] Basic command routing system
- [x] Database schema & models
- [x] Hyperliquid SDK integration

### Phase 2: Wallet Integration (Week 3)
- [x] QR Code + WalletConnect implementation
- [x] Multi-wallet button support (MetaMask, Trust, Rainbow, etc.)
- [x] Session-based connection management
- [x] Balance & position tracking via existing Hyperliquid SDK

### Phase 3: Grid Engine (Week 4-5)
- [x] Grid strategy implementation
- [x] Order management system
- [x] Risk management controls
- [x] Real-time monitoring

### Phase 4: User Experience (Week 6)
- [x] Interactive command wizards
- [x] Rich message formatting
- [x] Error handling & user feedback
- [x] Help documentation

### Phase 5: Advanced Features (Week 7-8)
- [x] Multi-grid management
- [x] Performance analytics
- [x] Referral system
- [x] XP/rewards system

### Phase 6: Production (Week 9-10)
- [x] Security audit
- [x] Load testing
- [x] Monitoring setup
- [x] Deployment automation

## 📁 Project File Structure
```
tg-bot/
├── src/
│   ├── bot/
│   │   ├── main.py               # Main bot entry
│   │   ├── middleware/           # Auth, logging, rate limiting
│   │   └── handlers/             # Command handlers
│   ├── wallet/
│   │   ├── connection.py         # QR Code + WalletConnect logic
│   │   ├── qr_generator.py       # QR code generation
│   │   ├── session_manager.py    # Session management
│   │   └── deep_links.py         # Mobile wallet deep links
│   ├── services/
│   │   ├── hyperliquid/          # Existing GridTrading integration
│   │   ├── database/             # DB operations (minimal)
│   │   ├── trading/              # Grid engine wrapper
│   │   └── notifications/        # Alert system
│   ├── models/                   # Database models
│   ├── utils/                    # Helper functions
│   └── config/                   # Configuration
├── tests/                        # Test suites
├── docs/                         # Documentation
├── scripts/                      # Deployment scripts
├── requirements.txt
├── .env.example
└── README.md
```

## 🔗 **Wallet Connection Implementation**

### **QR Code + Clickable Wallet Buttons Approach**

```python
# Example bot message with QR code and wallet buttons
async def show_wallet_connection(self, update, context):
    # Generate WalletConnect URI
    wc_uri = self.generate_walletconnect_uri(update.effective_user.id)
    
    # Create QR code image
    qr_image = self.create_qr_code(wc_uri)
    
    # Create wallet buttons (same URI, different formats)
    keyboard = [
        [
            InlineKeyboardButton("🦊 MetaMask", url=f"metamask://wc?uri={wc_uri}"),
            InlineKeyboardButton("🛡️ Trust Wallet", url=f"trust://wc?uri={wc_uri}")
        ],
        [
            InlineKeyboardButton("🌈 Rainbow", url=f"rainbow://wc?uri={wc_uri}"),
            InlineKeyboardButton("📱 Coinbase", url=f"cbwallet://wc?uri={wc_uri}")
        ],
        [
            InlineKeyboardButton("🌐 Browser Extension", url=f"https://metamask.app.link/wc?uri={wc_uri}")
        ],
        [
            InlineKeyboardButton("🔑 Enter Private Key", callback_data="private_key")
        ]
    ]
    
    await update.message.reply_photo(
        photo=qr_image,
        caption="🔗 **Connect Your Wallet**\n\n"
                "📷 **Scan QR code** with any wallet app\n"
                "OR\n"
                "📱 **Click your wallet** below:",
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode='Markdown'
    )
```

### **Universal Coverage:**
- ✅ **Desktop users:** Scan QR with mobile wallet OR click browser extension
- ✅ **Mobile users:** Tap wallet button OR scan QR code  
- ✅ **Any wallet:** Same WalletConnect protocol works everywhere
- ✅ **Fallback:** Private key input always available

### **Integration with Existing Grid Code:**
```python
# Seamlessly integrates with existing GridTrading class
session = wallet_handler.get_user_session(user_id)
address = session['address']
info = session['info'] 
exchange = session['exchange']

# Use existing GridTrading exactly as before
trading = GridTrading(address, info, exchange, coin, gridnum, ...)
trading.compute()  # No changes needed!
```

This architecture provides a robust, scalable foundation for the Telegram grid trading bot with universal wallet support, session-based security, and seamless integration with the existing Hyperliquid codebase.
