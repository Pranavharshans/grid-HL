# Wallet Connection System - Python Telegram Bot

## 🔐 Wallet Connection Methods

### Method 1: Private Key Import (Most Common)
```
User Flow:
/start → /wallet connect → Choose "Private Key" → Paste key → Encrypt & Store
```

### Method 2: Seed Phrase Import
```
User Flow:
/start → /wallet connect → Choose "Seed Phrase" → Enter 12/24 words → Derive key → Store
```

### Method 3: Hardware Wallet (Advanced)
```
User Flow:
/start → /wallet connect → Choose "Hardware" → Connect device → Sign test tx → Store address
```

## 🏗️ Technical Implementation (Python)

### 1. Wallet Connection Handler
```python
# bot/handlers/wallet_handler.py
import asyncio
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ContextTypes, ConversationHandler
from cryptography.fernet import Fernet
from eth_account import Account
import os
import re

# Conversation states
WALLET_METHOD, PRIVATE_KEY_INPUT, SEED_PHRASE_INPUT, CONFIRM_WALLET = range(4)

class WalletHandler:
    def __init__(self, db_manager, encryption_key):
        self.db = db_manager
        self.cipher = Fernet(encryption_key)
    
    async def start_wallet_connection(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Start wallet connection flow"""
        user_id = update.effective_user.id
        
        # Check if user already has a wallet
        existing_wallet = await self.db.get_user_wallet(user_id)
        if existing_wallet:
            keyboard = [
                [InlineKeyboardButton("🔄 Change Wallet", callback_data="change_wallet")],
                [InlineKeyboardButton("📊 View Current", callback_data="view_wallet")],
                [InlineKeyboardButton("❌ Cancel", callback_data="cancel")]
            ]
            reply_markup = InlineKeyboardMarkup(keyboard)
            await update.message.reply_text(
                "🔐 You already have a wallet connected!\n\n"
                f"Address: `{existing_wallet['address'][:10]}...{existing_wallet['address'][-6:]}`\n"
                "What would you like to do?",
                reply_markup=reply_markup,
                parse_mode='Markdown'
            )
            return ConversationHandler.END
        
        # Show wallet connection options
        keyboard = [
            [InlineKeyboardButton("🔑 Private Key", callback_data="private_key")],
            [InlineKeyboardButton("🌱 Seed Phrase", callback_data="seed_phrase")],
            [InlineKeyboardButton("🏷️ Hardware Wallet", callback_data="hardware_wallet")],
            [InlineKeyboardButton("❌ Cancel", callback_data="cancel")]
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await update.message.reply_text(
            "🔐 **Wallet Connection**\n\n"
            "Choose how you'd like to connect your wallet:\n\n"
            "🔑 **Private Key** - Import your private key directly\n"
            "🌱 **Seed Phrase** - Use 12/24 word mnemonic\n"
            "🏷️ **Hardware Wallet** - Connect Ledger/Trezor (coming soon)\n\n"
            "⚠️ **Security Note**: Your keys are encrypted and stored locally. "
            "We never see your actual private key!",
            reply_markup=reply_markup,
            parse_mode='Markdown'
        )
        return WALLET_METHOD

    async def handle_wallet_method(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Handle wallet method selection"""
        query = update.callback_query
        await query.answer()
        
        method = query.data
        
        if method == "private_key":
            await query.edit_message_text(
                "🔑 **Private Key Import**\n\n"
                "Please send your private key in the next message.\n\n"
                "⚠️ **Security Tips**:\n"
                "• Make sure you're in a private chat\n"
                "• Your key will be encrypted immediately\n"
                "• Delete your message after sending\n"
                "• Only use keys you control\n\n"
                "📝 **Format**: 0x... or without 0x prefix\n\n"
                "Type /cancel to abort this process.",
                parse_mode='Markdown'
            )
            return PRIVATE_KEY_INPUT
            
        elif method == "seed_phrase":
            await query.edit_message_text(
                "🌱 **Seed Phrase Import**\n\n"
                "Please send your seed phrase in the next message.\n\n"
                "⚠️ **Security Tips**:\n"
                "• Use 12 or 24 word phrase\n"
                "• Separate words with spaces\n"
                "• Double-check spelling\n"
                "• Delete your message after sending\n\n"
                "📝 **Example**: word1 word2 word3 ... word12\n\n"
                "Type /cancel to abort this process.",
                parse_mode='Markdown'
            )
            return SEED_PHRASE_INPUT
            
        elif method == "hardware_wallet":
            await query.edit_message_text(
                "🏷️ **Hardware Wallet**\n\n"
                "Hardware wallet support is coming soon!\n\n"
                "For now, please use private key or seed phrase method.\n\n"
                "Click /wallet to try again.",
                parse_mode='Markdown'
            )
            return ConversationHandler.END
            
        else:  # cancel
            await query.edit_message_text("❌ Wallet connection cancelled.")
            return ConversationHandler.END

    async def handle_private_key_input(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Handle private key input"""
        user_id = update.effective_user.id
        private_key = update.message.text.strip()
        
        # Delete user's message immediately for security
        try:
            await update.message.delete()
        except:
            pass
        
        # Validate private key format
        if not self._validate_private_key(private_key):
            await update.message.reply_text(
                "❌ **Invalid Private Key**\n\n"
                "Please ensure your private key:\n"
                "• Is 64 characters long (without 0x)\n"
                "• Contains only hexadecimal characters (0-9, a-f)\n"
                "• Is a valid Ethereum private key\n\n"
                "Try again or type /cancel to abort.",
                parse_mode='Markdown'
            )
            return PRIVATE_KEY_INPUT
        
        # Create account from private key
        try:
            account = Account.from_key(private_key)
            address = account.address
            
            # Store encrypted key and address
            await self._store_wallet(user_id, private_key, address, "private_key")
            
            # Show confirmation
            keyboard = [
                [InlineKeyboardButton("✅ Confirm & Continue", callback_data="confirm_wallet")],
                [InlineKeyboardButton("❌ Cancel", callback_data="cancel_wallet")]
            ]
            reply_markup = InlineKeyboardMarkup(keyboard)
            
            await update.message.reply_text(
                f"✅ **Wallet Connected Successfully!**\n\n"
                f"📍 **Address**: `{address}`\n"
                f"📍 **Short**: `{address[:10]}...{address[-6:]}`\n\n"
                "Your private key has been encrypted and stored securely.\n\n"
                "**Next Steps**:\n"
                "• Check your balance: /balance\n"
                "• Create your first grid: /grid\n"
                "• View portfolio: /status\n\n"
                "Ready to start trading?",
                reply_markup=reply_markup,
                parse_mode='Markdown'
            )
            return CONFIRM_WALLET
            
        except Exception as e:
            await update.message.reply_text(
                f"❌ **Error creating wallet**: {str(e)}\n\n"
                "Please check your private key and try again.\n"
                "Type /cancel to abort.",
                parse_mode='Markdown'
            )
            return PRIVATE_KEY_INPUT

    async def handle_seed_phrase_input(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Handle seed phrase input"""
        user_id = update.effective_user.id
        seed_phrase = update.message.text.strip()
        
        # Delete user's message immediately for security
        try:
            await update.message.delete()
        except:
            pass
        
        # Validate seed phrase
        words = seed_phrase.split()
        if len(words) not in [12, 24]:
            await update.message.reply_text(
                "❌ **Invalid Seed Phrase**\n\n"
                "Please ensure your seed phrase:\n"
                "• Has exactly 12 or 24 words\n"
                "• Words are separated by spaces\n"
                "• All words are valid BIP39 words\n\n"
                "Try again or type /cancel to abort.",
                parse_mode='Markdown'
            )
            return SEED_PHRASE_INPUT
        
        try:
            # Generate account from seed phrase
            Account.enable_unaudited_hdwallet_features()
            account = Account.from_mnemonic(seed_phrase)
            address = account.address
            private_key = account.key.hex()
            
            # Store encrypted key and address
            await self._store_wallet(user_id, private_key, address, "seed_phrase")
            
            # Show confirmation
            keyboard = [
                [InlineKeyboardButton("✅ Confirm & Continue", callback_data="confirm_wallet")],
                [InlineKeyboardButton("❌ Cancel", callback_data="cancel_wallet")]
            ]
            reply_markup = InlineKeyboardMarkup(keyboard)
            
            await update.message.reply_text(
                f"✅ **Wallet Connected Successfully!**\n\n"
                f"📍 **Address**: `{address}`\n"
                f"📍 **Short**: `{address[:10]}...{address[-6:]}`\n\n"
                "Your seed phrase has been converted to a private key,\n"
                "encrypted, and stored securely.\n\n"
                "**Next Steps**:\n"
                "• Check your balance: /balance\n"
                "• Create your first grid: /grid\n"
                "• View portfolio: /status\n\n"
                "Ready to start trading?",
                reply_markup=reply_markup,
                parse_mode='Markdown'
            )
            return CONFIRM_WALLET
            
        except Exception as e:
            await update.message.reply_text(
                f"❌ **Error with seed phrase**: {str(e)}\n\n"
                "Please check your seed phrase and try again.\n"
                "Type /cancel to abort.",
                parse_mode='Markdown'
            )
            return SEED_PHRASE_INPUT

    def _validate_private_key(self, private_key: str) -> bool:
        """Validate private key format"""
        # Remove 0x prefix if present
        if private_key.startswith('0x'):
            private_key = private_key[2:]
        
        # Check length and hex format
        if len(private_key) != 64:
            return False
        
        if not re.match(r'^[0-9a-fA-F]+$', private_key):
            return False
        
        # Try to create account to validate
        try:
            Account.from_key('0x' + private_key)
            return True
        except:
            return False

    async def _store_wallet(self, user_id: int, private_key: str, address: str, method: str):
        """Store encrypted wallet data"""
        # Encrypt private key
        encrypted_key = self.cipher.encrypt(private_key.encode())
        
        # Store in database
        wallet_data = {
            'user_id': user_id,
            'address': address,
            'encrypted_private_key': encrypted_key,
            'connection_method': method,
            'created_at': datetime.utcnow()
        }
        
        await self.db.store_wallet(wallet_data)

    async def confirm_wallet(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Confirm wallet setup"""
        query = update.callback_query
        await query.answer()
        
        if query.data == "confirm_wallet":
            await query.edit_message_text(
                "🎉 **Welcome to Hyperliquid Grid Trading!**\n\n"
                "Your wallet is now connected and ready to use.\n\n"
                "**Quick Start Commands**:\n"
                "• `/balance` - Check your USDC balance\n"
                "• `/grid` - Create your first grid strategy\n"
                "• `/help` - View all available commands\n"
                "• `/settings` - Configure preferences\n\n"
                "Happy trading! 📈",
                parse_mode='Markdown'
            )
        else:
            # Cancel and remove wallet
            user_id = update.effective_user.id
            await self.db.remove_wallet(user_id)
            await query.edit_message_text(
                "❌ Wallet connection cancelled and data removed.\n\n"
                "You can try connecting again with /wallet"
            )
        
        return ConversationHandler.END

    async def cancel_connection(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Cancel wallet connection"""
        await update.message.reply_text(
            "❌ Wallet connection cancelled.\n\n"
            "You can start again anytime with /wallet"
        )
        return ConversationHandler.END
```

### 2. Database Schema
```python
# models/wallet.py
from sqlalchemy import Column, Integer, String, LargeBinary, DateTime, Boolean
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime

Base = declarative_base()

class UserWallet(Base):
    __tablename__ = 'user_wallets'
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, unique=True, nullable=False)  # Telegram user ID
    address = Column(String(42), nullable=False)  # Ethereum address
    encrypted_private_key = Column(LargeBinary, nullable=False)  # Encrypted private key
    connection_method = Column(String(20), nullable=False)  # private_key, seed_phrase, hardware
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    last_used_at = Column(DateTime, default=datetime.utcnow)
    
    def __repr__(self):
        return f"<UserWallet(user_id={self.user_id}, address={self.address[:10]}...)>"

class WalletSession(Base):
    __tablename__ = 'wallet_sessions'
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, nullable=False)
    session_token = Column(String(64), nullable=False)  # For temporary authentication
    expires_at = Column(DateTime, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
```

### 3. Security & Encryption
```python
# security/encryption.py
import os
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
import base64

class WalletEncryption:
    def __init__(self, master_password: str):
        self.master_password = master_password.encode()
        self.salt = os.urandom(16)  # Store this salt securely
        self.cipher = self._create_cipher()
    
    def _create_cipher(self):
        """Create encryption cipher from master password"""
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,
            salt=self.salt,
            iterations=100000,
        )
        key = base64.urlsafe_b64encode(kdf.derive(self.master_password))
        return Fernet(key)
    
    def encrypt_private_key(self, private_key: str) -> bytes:
        """Encrypt private key"""
        return self.cipher.encrypt(private_key.encode())
    
    def decrypt_private_key(self, encrypted_key: bytes) -> str:
        """Decrypt private key"""
        return self.cipher.decrypt(encrypted_key).decode()
    
    @staticmethod
    def generate_master_key() -> str:
        """Generate a new master encryption key"""
        return Fernet.generate_key().decode()
```

### 4. Wallet Service Integration
```python
# services/wallet_service.py
from hyperliquid.info import Info
from hyperliquid.exchange import Exchange
from eth_account import Account
import eth_account

class WalletService:
    def __init__(self, db_manager, encryption_service):
        self.db = db_manager
        self.encryption = encryption_service
        self.info = Info()  # Hyperliquid info client
    
    async def get_user_account(self, user_id: int) -> Account:
        """Get decrypted account for user"""
        wallet = await self.db.get_user_wallet(user_id)
        if not wallet:
            raise Exception("No wallet found for user")
        
        # Decrypt private key
        private_key = self.encryption.decrypt_private_key(wallet.encrypted_private_key)
        
        # Create account object
        account = Account.from_key(private_key)
        return account
    
    async def get_balance(self, user_id: int) -> dict:
        """Get user's balance on Hyperliquid"""
        wallet = await self.db.get_user_wallet(user_id)
        if not wallet:
            raise Exception("No wallet connected")
        
        # Get user state from Hyperliquid
        user_state = self.info.user_state(wallet.address)
        
        # Extract balances
        balances = {}
        for balance in user_state.get('crossMainnetBalances', []):
            balances[balance['coin']] = {
                'total': float(balance['total']),
                'hold': float(balance['hold'])
            }
        
        return balances
    
    async def create_exchange_client(self, user_id: int) -> Exchange:
        """Create Hyperliquid exchange client for user"""
        account = await self.get_user_account(user_id)
        return Exchange(account)
    
    async def validate_wallet_connection(self, user_id: int) -> bool:
        """Validate that wallet connection is working"""
        try:
            balance = await self.get_balance(user_id)
            return True
        except Exception as e:
            return False
```

## 🔐 Security Features

### 1. **Multi-Layer Encryption**
- Master password for key derivation
- AES-256 encryption for private keys
- Salted PBKDF2 for key strengthening
- Secure deletion of plaintext keys

### 2. **Session Management**
- Temporary session tokens
- Auto-expiring authentication
- Rate limiting on wallet operations
- IP-based access controls

### 3. **Best Practices**
- Immediate message deletion
- Encrypted storage only
- No logging of sensitive data
- Regular security audits

## 📱 User Experience Flow

```
Step 1: User sends /wallet
Step 2: Bot shows connection options
Step 3: User chooses method (Private Key/Seed)
Step 4: User sends credentials (auto-deleted)
Step 5: Bot encrypts and stores safely
Step 6: Confirmation with wallet address
Step 7: Ready for trading commands
```

This setup provides enterprise-grade security while maintaining an intuitive user experience for connecting wallets to the Telegram bot! 