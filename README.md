# 🌊 TradeWave

**Ride Every Wave. Miss Nothing.**

A non-custodial Telegram crypto trading bot powered by [Coinbase Developer Platform](https://cdp.coinbase.com/).

## Features

✨ **Non-Custodial** — Users own their wallets; the bot has zero access to funds

💰 **Real-Time Trading** — Buy/sell ETH, SOL, and more at market rates via Coinbase CDP

🔐 **Encrypted Storage** — Wallet seeds encrypted with AES-256-GCM before storage

📱 **Telegram Native** — Full multi-step trading flows directly in Telegram

⚡ **Fast Withdrawals** — Send crypto to any external wallet in seconds

## Quick Start

### 1. Prerequisites

- Node.js v18+
- npm v9+
- A Telegram Bot Token (from [@BotFather](https://t.me/botfather))
- Coinbase Developer Platform credentials ([portal.cdp.coinbase.com](https://portal.cdp.coinbase.com))

### 2. Installation

```bash
git clone https://github.com/vincoradesign-pixel/sturdy-telegram.git
cd sturdy-telegram
npm install
```

### 3. Configuration

Copy `.env.example` to `.env` and fill in your credentials:

```bash
cp .env.example .env
```

Then edit `.env` with:

```env
TELEGRAM_BOT_TOKEN=your_token_from_botfather
CDP_API_KEY_NAME=your_cdp_key
CDP_PRIVATE_KEY=your_cdp_private_key
WALLET_ENCRYPTION_KEY=<generate with: node -e "console.log(require('crypto').randomBytes(32).toString('hex'))">
NETWORK_ID=base-sepolia    # Use base-sepolia for testing
```

### 4. Run

```bash
npm start
```

You should see:
```
🌊 TradeWave starting…
✅ Coinbase CDP configured (network: base-sepolia)
🌊 TradeWave live as @YourBotName (network: base-sepolia)
```

## Commands

| Command | Description |
|---------|-------------|
| `/start` | Initialize wallet & welcome |
| `/balance` | View all asset balances |
| `/deposit` | Show deposit address |
| `/buy` | Buy crypto with USDC |
| `/sell` | Sell crypto for USDC |
| `/withdraw` | Send crypto to external wallet |
| `/portfolio` | Full portfolio overview |
| `/help` | Show all commands |

## Architecture

```
Telegram User
     ↓
  Grammy Bot (Telegram API interface)
     ↓
  Bot.js (Command routing, multi-step flows)
     ↓
  Wallet.js (Coinbase CDP integration)
     ↓
  Encryption.js + DB.js (Secure storage)
     ↓
  Blockchain (Base / Ethereum)
```

## Security

- **Non-Custodial**: TradeWave never holds user private keys. Each user owns their wallet.
- **AES-256-GCM**: Wallet seeds encrypted at rest with authenticated encryption.
- **Environment Secrets**: All credentials stored in `.env` (never committed to repo).
- **Graceful Shutdown**: Handles SIGINT/SIGTERM cleanly.

## Development

### Testing on Base Sepolia (Testnet)

1. Set `NETWORK_ID=base-sepolia` in `.env`
2. Get free test ETH from [Base Sepolia Faucet](https://faucet.quicknode.com/base/sepolia)
3. Deploy a test USDC contract or use an existing one on Sepolia

### Extending TradeWave

See [`DEVELOPER_DOCS.md`](./DEVELOPER_DOCS.md) for:

- Adding new assets
- Implementing price alerts
- Setting up Dollar-Cost Averaging (DCA)
- Production deployment guides
- Troubleshooting

## Deployment

### Replit (Recommended for Dev)

1. Fork this repo to Replit
2. Add environment variables in **Secrets** (🔐)
3. Click **Run** — bot starts automatically
4. Enable **Always On** for 24/7 uptime

### VPS / Cloud (Production)

```bash
# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Clone & install
git clone https://github.com/vincoradesign-pixel/sturdy-telegram.git
cd sturdy-telegram
npm install --production

# Set env vars (use secrets manager)
export TELEGRAM_BOT_TOKEN=...
export CDP_API_KEY_NAME=...
export CDP_PRIVATE_KEY=...
export WALLET_ENCRYPTION_KEY=...

# Run with PM2
npm install -g pm2
pm2 start index.js --name tradewave
pm2 startup
pm2 save
```

## Documentation

For a complete technical reference, see [`DEVELOPER_DOCS.md`](./DEVELOPER_DOCS.md).

## License

MIT

## Support

Having issues? Check the [Troubleshooting](./DEVELOPER_DOCS.md#12-troubleshooting) section in the developer docs.

---

**Built with ❤️ for crypto traders everywhere.**