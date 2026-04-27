# 🌊 TradeWave Developer Documentation

**Version 1.0 · February 2025**

Complete technical reference for the TradeWave Telegram crypto trading bot.

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Prerequisites & Installation](#2-prerequisites--installation)
3. [Environment Configuration](#3-environment-configuration)
4. [Core Modules Walkthrough](#4-core-modules-walkthrough)
5. [Coinbase CDP Integration](#5-coinbase-cdp-integration)
6. [Security Model](#6-security-model)
7. [Bot Commands & Flows](#7-bot-commands--user-flows)
8. [API Reference](#8-internal-api-reference)
9. [Error Handling](#9-error-handling)
10. [Extending TradeWave](#10-extending-tradewave)
11. [Deployment Guide](#11-deployment-guide)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Overview & Architecture

TradeWave is a non-custodial Telegram trading bot that allows users to manage on-chain crypto wallets and execute trades directly from Telegram. It is built on the Coinbase Developer Platform (CDP) SDK and uses Grammy.js for Telegram bot interaction.

### 1.1 Key Capabilities

- Non-custodial wallet creation per user (one wallet per Telegram user ID)
- Real-time balance fetching for ETH, BTC, USDC, SOL
- Market-rate buy and sell swaps via Coinbase CDP trade API
- Crypto withdrawals to arbitrary external addresses
- AES-256-GCM encrypted wallet seed storage
- Multi-step conversational flows within Telegram

### 1.2 Architecture Diagram

```
User (Telegram)
       ↓
  Grammy Bot Layer
       ↓
  CDP SDK
       ↓
  Blockchain (Base / Ethereum)
       ↓
  Encrypted Storage (SQLite DB)
```

### 1.3 Technology Stack

| Layer | Technology | Purpose |
|-------|------------|----------|
| Telegram Interface | Grammy.js 1.21+ | Bot framework, inline keyboards, message routing |
| Trading Engine | Coinbase CDP SDK 0.10+ | Wallet management, asset swaps, transfers |
| Wallet Encryption | Node.js crypto (AES-256-GCM) | Encrypt seeds at rest before DB storage |
| Database | better-sqlite3 | Local encrypted wallet data |
| Runtime | Node.js 18+ | JavaScript runtime environment |

---

## 2. Prerequisites & Installation

### 2.1 Requirements

- Node.js v18.0.0 or higher
- npm v9.0.0 or higher
- A Telegram Bot Token (from @BotFather)
- A Coinbase Developer Platform account and API credentials

### 2.2 Clone & Install

```bash
git clone https://github.com/vincoradesign-pixel/sturdy-telegram.git
cd sturdy-telegram
npm install
```

### 2.3 Dependencies

| Package | Version | Description |
|---------|---------|-------------|
| grammy | ^1.21.1 | Telegram Bot API framework |
| @coinbase/coinbase-sdk | ^0.10.0 | Coinbase Developer Platform SDK |
| better-sqlite3 | ^9.2.0 | SQLite database driver |
| dotenv | ^16.3.1 | Environment variable loader |

---

## 3. Environment Configuration

All sensitive credentials are loaded from environment variables.

### 3.1 Required Variables

| Variable | Description | Source |
|----------|-------------|--------|
| TELEGRAM_BOT_TOKEN | Telegram bot API token | @BotFather on Telegram |
| CDP_API_KEY_NAME | Coinbase CDP API key name | portal.cdp.coinbase.com |
| CDP_PRIVATE_KEY | Coinbase CDP private key (PEM format) | portal.cdp.coinbase.com |
| WALLET_ENCRYPTION_KEY | 32-byte hex string for AES-256-GCM | Generated via command below |
| NETWORK_ID | Target blockchain network | base-mainnet \| base-sepolia \| ethereum-mainnet |

### 3.2 Generating the Encryption Key

Run this one-time command to generate a secure 32-byte encryption key:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

⚠️ **Security Warning**: Store the encryption key ONLY in your environment secrets manager. If exposed, all user wallets are compromised.

### 3.3 .env File Template

```env
TELEGRAM_BOT_TOKEN=your_bot_token_from_botfather
CDP_API_KEY_NAME=your_cdp_key_name
CDP_PRIVATE_KEY=your_cdp_private_key
WALLET_ENCRYPTION_KEY=your_64_char_hex_string
NETWORK_ID=base-sepolia
```

Add `.env` to `.gitignore`:

```bash
echo '.env' >> .gitignore
```

---

## 4. Core Modules Walkthrough

### 4.1 Initialization

The bot initializes core clients at startup:

```javascript
const bot = new Bot(TELEGRAM_TOKEN);
Coinbase.configure({
  apiKeyName: CDP_API_KEY_NAME,
  privateKey: CDP_PRIVATE_KEY,
});
```

### 4.2 Encryption Module (encryption.js)

Wallet seeds are encrypted using AES-256-GCM:

```javascript
function encrypt(text) {
  const iv      = crypto.randomBytes(16);
  const cipher  = crypto.createCipheriv('aes-256-gcm', KEY_BUF, iv);
  const enc     = Buffer.concat([cipher.update(text, 'utf8'), cipher.final()]);
  const authTag = cipher.getAuthTag();
  return `${iv.toString('hex')}:${authTag.toString('hex')}:${enc.toString('hex')}`;
}
```

**Why AES-256-GCM?** GCM mode provides authenticated encryption — the auth tag detects tampering before decryption.

### 4.3 Wallet Management

Each Telegram user ID maps to exactly one wallet:

```javascript
async function getOrCreateWallet(userId) {
  const stored = walletOps.get(userId);
  if (stored) {
    const walletData = JSON.parse(decrypt(stored));
    return await Wallet.import(walletData);
  }
  const wallet   = await Wallet.create({ networkId: NETWORK_ID });
  const exported = wallet.export();
  walletOps.set(userId, encrypt(JSON.stringify(exported)));
  return wallet;
}
```

### 4.4 Session Management

Multi-step flows use an in-memory session Map:

```javascript
const sessions = new Map();

function getSession(userId) {
  if (!sessions.has(userId)) sessions.set(userId, {});
  return sessions.get(userId);
}
```

> **Note**: In-memory sessions are lost on restart. For production, persist to database.

---

## 5. Coinbase CDP Integration

### 5.1 Supported Operations

| Operation | CDP Method | Notes |
|-----------|------------|-------|
| Create wallet | `Wallet.create({ networkId })` | Creates a new HD wallet |
| Import wallet | `Wallet.import(walletData)` | Reconstructs from exported seed |
| Get address | `wallet.getDefaultAddress()` | Returns primary address |
| Get balance | `wallet.getBalance(assetId)` | Returns balance as string |
| Transfer | `wallet.createTransfer({...})` | Send crypto to external address |
| Trade | `wallet.createTrade({...})` | Swap between assets at market rate |

### 5.2 Asset ID Convention

The CDP SDK uses lowercase asset IDs:

```javascript
// ✅ Correct
await wallet.getBalance('eth');
await wallet.createTrade({ fromAssetId: 'usdc', toAssetId: 'eth' });

// ❌ Wrong
await wallet.getBalance('ETH');
```

### 5.3 Executing a Trade

```javascript
const trade = await wallet.createTrade({
  amount:      100,        // USDC amount to spend
  fromAssetId: 'usdc',
  toAssetId:   'eth',
});

await trade.wait();
const received = trade.getToAmount();
const txHash   = trade.getTransaction().getTransactionHash();
```

### 5.4 Executing a Transfer

```javascript
const tx = await wallet.createTransfer({
  amount:      0.05,
  assetId:     'eth',
  destination: '0xRecipientAddress...',
});

await tx.wait();
const txHash = tx.getTransactionHash();
```

---

## 6. Security Model

### 6.1 Non-Custodial Architecture

TradeWave never holds user funds. All wallets are user-owned and created on-chain via CDP SDK. The bot operator cannot move user funds without the encrypted seed.

### 6.2 Threat Model

| Threat | Mitigation | Risk Level |
|--------|------------|------------|
| DB breach — seed exposure | Seeds encrypted with AES-256-GCM | Low (if key is secure) |
| Encryption key exposure | Key stored only in env secrets | Critical |
| Bot token theft | Revoke via @BotFather | Medium |
| Man-in-the-middle | All API calls are TLS-encrypted | Low |
| Replay attacks | GCM auth tag + random IV | Low |

### 6.3 Security Checklist

- [ ] Use environment secrets for all sensitive values
- [ ] Rotate CDP API key and encryption key on compromise
- [ ] Set `NODE_ENV=production` in production
- [ ] Run `npm audit` regularly
- [ ] Validate all user inputs (addresses, amounts)
- [ ] Never log wallet seeds or private keys

---

## 7. Bot Commands & User Flows

### 7.1 Commands

| Command | Handler | Description |
|---------|---------|-------------|
| `/start` | `bot.command('start')` | Welcome, creates wallet |
| `/help` | `bot.command('help')` | Show all commands |
| `/balance` | `bot.command('balance')` | Display asset balances |
| `/deposit` | `bot.command('deposit')` | Show wallet address |
| `/portfolio` | `bot.command('portfolio')` | Full holdings overview |
| `/buy` | `bot.command('buy')` | Initiate buy flow |
| `/sell` | `bot.command('sell')` | Initiate sell flow |
| `/withdraw` | `bot.command('withdraw')` | Initiate withdraw flow |

### 7.2 Multi-Step Flow: Withdraw Example

```
Step 1: User taps 'Withdraw'
        → session.step = 'withdraw:select_asset'

Step 2: User selects asset (e.g., ETH)
        → session.withdrawAsset = 'ETH'
        → session.step = 'withdraw:enter_address'

Step 3: User types destination address
        → Validated via regex /^0x[a-fA-F0-9]{40}$/
        → session.withdrawAddress = '0x...'
        → session.step = 'withdraw:enter_amount'

Step 4: User types amount
        → session.withdrawAmount = parseFloat(text)
        → session.step = 'withdraw:confirm'

Step 5: User confirms → executeWithdraw(userId, asset, amount, address)
        → Session cleared
```

---

## 8. Internal API Reference

### 8.1 getOrCreateWallet(userId)

Returns the CDP Wallet object for the given Telegram user ID.

```javascript
const wallet = await getOrCreateWallet(userId);
```

**Parameters:**
- `userId` (number): Telegram user ID from `ctx.from.id`

**Returns:** `Promise<Wallet>`

### 8.2 getAddress(userId)

Fetches the user's wallet address.

```javascript
const address = await getAddress(userId);
console.log(address); // '0x...'
```

**Returns:** `Promise<string>` (Ethereum address)

### 8.3 getBalances(userId)

Fetches balances for ETH, USDC, and SOL.

```javascript
const balances = await getBalances(userId);
console.log(balances);
// { eth: '1.5', usdc: '100', sol: '0.5' }
```

**Returns:** `Promise<Object>` (asset -> balance map)

### 8.4 executeTrade(userId, fromAsset, toAsset, amount)

Executes a swap between two assets.

```javascript
const result = await executeTrade(userId, 'usdc', 'eth', 50);
console.log(result);
// { received: '0.025', txHash: '0x...' }
```

**Parameters:**
- `fromAsset` (string): Source asset (lowercase, e.g., 'usdc')
- `toAsset` (string): Target asset (lowercase, e.g., 'eth')
- `amount` (number): Quantity of fromAsset to spend

**Returns:** `Promise<{received, txHash}>`

### 8.5 executeWithdraw(userId, asset, amount, destination)

Sends crypto to an external address.

```javascript
const result = await executeWithdraw(userId, 'eth', 0.05, '0x...');
console.log(result);
// { txHash: '0x...' }
```

**Parameters:**
- `asset` (string): Asset to send (lowercase, e.g., 'eth')
- `amount` (number): Amount to send
- `destination` (string): Recipient Ethereum address

**Returns:** `Promise<{txHash}>`

### 8.6 encrypt(text) / decrypt(payload)

Encrypt/decrypt strings using AES-256-GCM.

```javascript
const encrypted = encrypt('sensitive data');
const decrypted = decrypt(encrypted);
```

**Returns:** 
- `encrypt`: `string` (colon-separated IV:authTag:ciphertext)
- `decrypt`: `string` (original plaintext)

---

## 9. Error Handling

### 9.1 Strategy

- All CDP operations are wrapped in try/catch
- Failed operations clear session state
- Users see friendly error messages
- Technical errors logged server-side only

### 9.2 Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| Insufficient balance | Trade amount > balance | Check balance; show user current balance |
| Invalid address | Malformed Ethereum address | Validate with regex before proceeding |
| CDP API failure | Network issue or rate limit | Retry with exponential backoff |
| Decryption failure | Wallet data corrupted | Indicates key rotation or DB corruption |
| Wallet not found | DB get() returns null | Trigger wallet creation flow |

---

## 10. Extending TradeWave

### 10.1 Adding a New Asset

Update the asset list in `wallet.js`:

```javascript
const assets = ['eth', 'usdc', 'sol', 'doge'];
```

Update buy/sell commands in `bot.js`:

```javascript
const kb = new InlineKeyboard()
  .text('ETH', 'buy_asset:ETH')
  .text('SOL', 'buy_asset:SOL')
  .text('DOGE', 'buy_asset:DOGE')
  .row()
  .text('❌ Cancel', 'cancel');
```

### 10.2 Adding Price Alerts

Polling-based price alerts:

```javascript
setInterval(async () => {
  const price = await fetchPrice('ETH');
  for (const [userId, prefs] of alertPrefs) {
    if (price < prefs.alertBelow) {
      await bot.api.sendMessage(userId, `📢 ETH is below $${prefs.alertBelow}!`);
    }
  }
}, 60_000);
```

### 10.3 Adding Dollar-Cost Averaging (DCA)

Store DCA schedules in the database:

```javascript
// Store DCA config
db.set(`dca:${userId}`, JSON.stringify({
  asset: 'ETH',
  amount: 50,
  intervalMs: 86400000, // 24 hours
}));

// Process on cron interval
setInterval(async () => {
  const users = await db.list('dca:');
  for (const key of users.keys) {
    const dca = JSON.parse(await db.get(key));
    const userId = key.replace('dca:', '');
    const wallet = await getOrCreateWallet(userId);
    await wallet.createTrade({
      amount: dca.amount,
      fromAssetId: 'usdc',
      toAssetId: dca.asset.toLowerCase(),
    });
  }
}, 3600_000);
```

---

## 11. Deployment Guide

### 11.1 Replit (Recommended for Development)

1. Fork repo to Replit
2. Open **Secrets** (🔐) and add all environment variables
3. Click **Run** — bot starts with long-polling automatically
4. Enable **Always On** for 24/7 uptime

### 11.2 VPS / Cloud (Production)

```bash
# Install Node.js 18+
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Clone and install
git clone https://github.com/vincoradesign-pixel/sturdy-telegram.git
cd sturdy-telegram
npm install --production

# Set environment variables
export TELEGRAM_BOT_TOKEN=...
export CDP_API_KEY_NAME=...
export CDP_PRIVATE_KEY=...
export WALLET_ENCRYPTION_KEY=...
export NETWORK_ID=base-mainnet

# Run with PM2 for process management
npm install -g pm2
pm2 start index.js --name tradewave
pm2 startup
pm2 save
```

### 11.3 Supported Networks

| Network ID | Type | Recommended Use |
|----------|------|----------|
| base-sepolia | Testnet | Development and testing (free faucet) |
| base-mainnet | Mainnet | Production — real funds on Base L2 |
| ethereum-mainnet | Mainnet | Production — real funds on Ethereum |

> Always test on base-sepolia first. Get free test ETH from [Base Sepolia Faucet](https://faucet.quicknode.com/base/sepolia).

---

## 12. Troubleshooting

### 12.1 Common Issues

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Bot doesn't respond | Missing/wrong TELEGRAM_BOT_TOKEN | Verify token in @BotFather |
| CDP API 401 error | Invalid CDP credentials | Check portal.cdp.coinbase.com |
| Decryption error | Wrong WALLET_ENCRYPTION_KEY | Ensure key matches wallet creation |
| 'Insufficient funds' | No USDC in wallet | Deposit USDC or use faucet |
| Trade fails | Wrong NETWORK_ID for asset | Check CDP docs for asset availability |
| Bot stops after idle | No keep-alive / process manager | Use PM2 or Replit Always On |

### 12.2 Useful Debug Commands

```bash
# Check environment variables
node -e "console.log(process.env.NETWORK_ID)"

# Validate CDP credentials
node -e "const {Coinbase}=require('@coinbase/coinbase-sdk'); Coinbase.configure({apiKeyName:process.env.CDP_API_KEY_NAME, privateKey:process.env.CDP_PRIVATE_KEY}); console.log('CDP OK')"

# Check database
node -e "const db=require('better-sqlite3'); const d=new db('./data/tradewave.db'); console.log(d.prepare('SELECT COUNT(*) as count FROM wallets').get())"
```

---

**TradeWave Developer Documentation · v1.0 · February 2025**

For support, reach out or open an issue on GitHub.

© 2025 TradeWave. All rights reserved.