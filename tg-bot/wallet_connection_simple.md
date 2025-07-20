# Simple Wallet Connection - Following Hyperliquid Patterns

## 🔍 **How Existing Hyperliquid Code Works**

Looking at the existing codebase, wallet connection is much simpler:

```python
# From example_utils.py - This is how ALL examples connect wallets:
def setup(base_url=None, skip_ws=False):
    # 1. Load config from JSON file
    config_path = os.path.join(os.path.dirname(__file__), "config.json")
    with open(config_path) as f:
        config = json.load(f)
    
    # 2. Create account directly from private key
    account: LocalAccount = eth_account.Account.from_key(config["secret_key"])
    address = config["account_address"]
    if address == "":
        address = account.address
    
    # 3. Create exchange client
    exchange = Exchange(account, base_url, account_address=address)
    return address, info, exchange
```

## 🎯 **Simplified Telegram Bot Approach**

### Method 1: Direct Private Key (Matches Existing Pattern)
```python
# bot/handlers/wallet_handler.py
import eth_account
from eth_account.signers.local import LocalAccount
from hyperliquid.exchange import Exchange
from hyperliquid.info import Info
from hyperliquid.utils import constants

class SimpleWalletHandler:
    def __init__(self, db_manager):
        self.db = db_manager
        self.user_sessions = {}  # In-memory wallet sessions

    async def connect_wallet_simple(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Simple wallet connection - just like the examples"""
        user_id = update.effective_user.id
        
        await update.message.reply_text(
            "🔑 **Connect Your Wallet**\n\n"
            "Send me your private key and I'll connect it immediately.\n\n"
            "⚠️ **Security**: Your key is used only for this session and never stored permanently.\n"
            "📝 **Format**: 0x... or without 0x prefix\n\n"
            "Type /cancel to abort.",
            parse_mode='Markdown'
        )
        return PRIVATE_KEY_INPUT

    async def handle_private_key(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Handle private key input - create session immediately"""
        user_id = update.effective_user.id
        private_key = update.message.text.strip()
        
        # Delete user's message for security
        try:
            await update.message.delete()
        except:
            pass
        
        try:
            # Create account (exactly like examples do)
            account: LocalAccount = eth_account.Account.from_key(private_key)
            address = account.address
            
            # Create Hyperliquid clients (exactly like examples do)
            info = Info(constants.MAINNET_API_URL, skip_ws=False)
            exchange = Exchange(account, constants.MAINNET_API_URL, account_address=address)
            
            # Test connection by getting user state
            user_state = info.user_state(address)
            balances = user_state.get("crossMainnetBalances", [])
            
            # Store session (temporary, not persistent)
            self.user_sessions[user_id] = {
                'account': account,
                'address': address,
                'info': info,
                'exchange': exchange,
                'connected_at': time.time()
            }
            
            # Show success message with balance
            usdc_balance = "0"
            for balance in balances:
                if balance['coin'] == 'USDC':
                    usdc_balance = balance['total']
                    break
            
            await update.message.reply_text(
                f"✅ **Wallet Connected!**\n\n"
                f"📍 **Address**: `{address[:10]}...{address[-6:]}`\n"
                f"💰 **USDC Balance**: {usdc_balance}\n\n"
                "**Available Commands**:\n"
                "• `/balance` - View full balance\n"
                "• `/grid` - Create grid strategy\n"
                "• `/status` - Check positions\n"
                "• `/disconnect` - Disconnect wallet\n\n"
                "Ready to trade! 🚀",
                parse_mode='Markdown'
            )
            return ConversationHandler.END
            
        except Exception as e:
            await update.message.reply_text(
                f"❌ **Connection Failed**: {str(e)}\n\n"
                "Please check your private key and try again.\n"
                "Type /cancel to abort.",
                parse_mode='Markdown'
            )
            return PRIVATE_KEY_INPUT

    def get_user_session(self, user_id: int):
        """Get user's wallet session"""
        return self.user_sessions.get(user_id)

    async def disconnect_wallet(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Disconnect wallet session"""
        user_id = update.effective_user.id
        
        if user_id in self.user_sessions:
            del self.user_sessions[user_id]
            await update.message.reply_text(
                "🔓 **Wallet Disconnected**\n\n"
                "Your wallet session has been cleared.\n"
                "Use /wallet to reconnect anytime."
            )
        else:
            await update.message.reply_text(
                "❌ No wallet connected.\n"
                "Use /wallet to connect."
            )
```

### Method 2: Agent/API Wallet Pattern (From Examples)
```python
async def connect_as_agent(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Connect using agent pattern - like basic_agent.py example"""
    user_id = update.effective_user.id
    
    # User provides main account private key first
    main_private_key = update.message.text.strip()
    
    try:
        # Create main account
        main_account: LocalAccount = eth_account.Account.from_key(main_private_key)
        main_address = main_account.address
        
        # Create exchange for main account
        main_exchange = Exchange(main_account, constants.MAINNET_API_URL)
        
        # Create agent (like basic_agent.py does)
        approve_result, agent_key = main_exchange.approve_agent()
        
        if approve_result["status"] != "ok":
            raise Exception(f"Agent approval failed: {approve_result}")
        
        # Create agent account
        agent_account: LocalAccount = eth_account.Account.from_key(agent_key)
        
        # Create agent exchange (can trade, but can't withdraw/transfer)
        agent_exchange = Exchange(
            agent_account, 
            constants.MAINNET_API_URL, 
            account_address=main_address
        )
        
        # Store agent session (safer - agent can't withdraw funds)
        self.user_sessions[user_id] = {
            'account': agent_account,
            'address': main_address,  # This is the main account address
            'info': Info(constants.MAINNET_API_URL, skip_ws=False),
            'exchange': agent_exchange,  # Agent exchange for trading
            'is_agent': True,
            'connected_at': time.time()
        }
        
        await update.message.reply_text(
            f"✅ **Agent Wallet Connected!**\n\n"
            f"📍 **Account**: `{main_address[:10]}...{main_address[-6:]}`\n"
            f"🤖 **Agent**: `{agent_account.address[:10]}...{agent_account.address[-6:]}`\n\n"
            "**Security**: Agent can trade but cannot withdraw funds.\n"
            "This is safer for bot usage!\n\n"
            "Ready to create grids! 🚀",
            parse_mode='Markdown'
        )
        
    except Exception as e:
        await update.message.reply_text(f"❌ Agent setup failed: {str(e)}")
```

### Method 3: Config File Pattern (For Advanced Users)
```python
async def connect_via_config(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Connect using config file pattern - like existing examples"""
    user_id = update.effective_user.id
    
    await update.message.reply_text(
        "📋 **Config File Connection**\n\n"
        "Send me a JSON config file with this format:\n\n"
        "```json\n"
        "{\n"
        '  "secret_key": "0x...",\n'
        '  "account_address": "0x..." // optional\n'
        "}\n"
        "```\n\n"
        "This follows the same pattern as Hyperliquid examples.",
        parse_mode='Markdown'
    )
    return CONFIG_FILE_INPUT

async def handle_config_file(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle config file input"""
    user_id = update.effective_user.id
    config_text = update.message.text.strip()
    
    try:
        # Parse JSON config (exactly like example_utils.py)
        config = json.loads(config_text)
        
        # Create account from config (exactly like examples)
        account: LocalAccount = eth_account.Account.from_key(config["secret_key"])
        address = config.get("account_address", "")
        if address == "":
            address = account.address
        
        # Create clients (exactly like examples)
        info = Info(constants.MAINNET_API_URL, skip_ws=False)
        exchange = Exchange(account, constants.MAINNET_API_URL, account_address=address)
        
        # Store session
        self.user_sessions[user_id] = {
            'account': account,
            'address': address,
            'info': info,
            'exchange': exchange,
            'connected_at': time.time()
        }
        
        await update.message.reply_text(
            f"✅ **Config Loaded Successfully!**\n\n"
            f"📍 **Address**: `{address[:10]}...{address[-6:]}`\n\n"
            "Wallet connected using config pattern! 🚀"
        )
        
    except Exception as e:
        await update.message.reply_text(f"❌ Config error: {str(e)}")
```

## 🔄 **Grid Trading Integration** 

```python
# services/grid_service.py
from hyperliquid.grid_trading import GridTrading

class GridService:
    def __init__(self, wallet_handler):
        self.wallet_handler = wallet_handler

    async def create_grid(self, user_id: int, coin: str, grid_config: dict):
        """Create grid using existing GridTrading class"""
        
        # Get user's wallet session
        session = self.wallet_handler.get_user_session(user_id)
        if not session:
            raise Exception("No wallet connected. Use /wallet first.")
        
        # Extract session data (exactly like Grid.py does)
        address = session['address']
        info = session['info']
        exchange = session['exchange']
        
        # Create GridTrading instance (exactly like Grid.py does)
        trading = GridTrading(
            address=address,
            info=info,
            exchange=exchange,
            COIN=coin,
            gridnum=grid_config["GRIDNUM"],
            gridmax=grid_config.get("GRIDMAX"),
            gridmin=grid_config.get("GRIDMIN"),
            tp=grid_config["TP"],
            eachgridamount=grid_config["EACHGRIDAMOUNT"],
            hasspot=grid_config.get("HASS_SPOT", False),
            enable_long_grid=grid_config.get("enable_long_grid", True),
            enable_short_grid=grid_config.get("enable_short_grid", False)
        )
        
        # Start the grid (exactly like Grid.py does)
        trading.compute()
        
        # Store grid instance for monitoring
        if not hasattr(self, 'user_grids'):
            self.user_grids = {}
        
        if user_id not in self.user_grids:
            self.user_grids[user_id] = []
        
        self.user_grids[user_id].append({
            'grid': trading,
            'coin': coin,
            'created_at': time.time(),
            'status': 'active'
        })
        
        return trading
```

## 🎯 **Why This Approach is Better**

### **1. Follows Existing Patterns**
- Uses exact same `eth_account.Account.from_key()` pattern
- Creates `Exchange` and `Info` clients exactly like examples
- Compatible with existing `GridTrading` class

### **2. Simpler Code**
- No complex encryption/decryption
- No database schemas for wallet storage
- Session-based (cleared when bot restarts)

### **3. More Secure**
- Keys only exist in memory during session
- No persistent storage of private keys
- Agent pattern provides additional security

### **4. Direct Integration**
- Drop-in replacement for existing `setup()` function
- Works with existing `GridTrading` class unchanged
- Same error handling and validation

## 📱 **User Flow**

```
1. User: /wallet
2. Bot: "Send your private key"
3. User: 0x123...
4. Bot: Connects immediately (like examples)
5. User: /grid BTC
6. Bot: Creates grid using existing GridTrading class
7. Grid runs exactly like standalone version
```

This approach is **much simpler** and follows the **exact patterns** used by all the existing Hyperliquid examples! No need for complex encryption or persistent storage - just session-based wallet connections that work exactly like the existing codebase. 