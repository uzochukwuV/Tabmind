# Issue #1 — Project Setup Guide
> Arbmind: CEX-DEX Arbitrage Agent with On-Chain Identity
> Based on the official lablab.ai tutorial with all Kraken team corrections applied

---

## Before You Start — Get All Your Credentials

You need **6 things**. All are free. Do these in parallel while reading the rest of this guide.

### 1. Kraken Account + API Key
> **Used for:** live/paper trading via Kraken CLI

1. Go to [kraken.com](https://kraken.com) and create an account
2. Complete **Intermediate KYC** (required for API access) — upload ID, takes ~10 min
3. Once verified, go to: **Settings → Security → API → Create API Key**
4. Configure permissions:
   - ✅ `Query Funds`
   - ✅ `Query Open Orders & Trades`
   - ✅ `Modify Orders` (needed to place orders)
   - ✅ `Cancel/Close Orders`
   - ❌ Do NOT enable `Withdraw Funds` — not needed, security risk
5. Copy and save `API Key` and `Private Key` — you only see the private key once

> **Note:** Paper trading (`kraken paper`) needs NO API key. You only need one for live trading.
> For the hackathon, start paper-only. Enable the key only if you want live fills.

---

### 2. PRISM API Key (Algo-Nomics)
> **Used for:** real-time prices, Spotter signals, SPARC risk scores across 50+ exchanges

1. Go to [prism.algo-nomics.com](https://prism.algo-nomics.com)
2. Sign up for a developer account
3. Navigate to **Settings → API Keys → Generate Key**
4. Copy your key — free tier gives generous daily call limits, no upgrade needed

---

### 3. Anthropic API Key
> **Used for:** Claude strategy router (selects arb vs funding vs signal strategy each loop)

1. Go to [console.anthropic.com](https://console.anthropic.com)
2. Create an account — you get free credits on signup
3. Go to **API Keys → Create Key**
4. Copy your key

---

### 4. Web3 RPC URL (Base Sepolia Testnet)
> **Used for:** calling ERC-8004 registry contracts, writing attestations on-chain

**Option A — Alchemy (recommended):**
1. Go to [alchemy.com](https://alchemy.com) → Create free account
2. Create a new app → Select **Base** → **Base Sepolia** network
3. Copy the HTTPS endpoint — looks like:
   `https://base-sepolia.g.alchemy.com/v2/YOUR_KEY`

**Option B — Public RPC (no signup, less reliable):**
```
https://sepolia.base.org
```

---

### 5. Agent Wallet Private Key
> **Used for:** signing EIP-712 trade intent attestations + registering on ERC-8004

Generate a fresh wallet locally — never reuse an existing personal wallet:

```bash
# Install Foundry first (one-time):
curl -L https://foundry.paradigm.xyz | bash
foundryup

# Generate a new wallet:
cast wallet new
```

Output looks like:
```
Successfully created new keypair.
Address:     0xABCD...1234
Private key: 0x1234...ABCD
```

Save both. The address is your agent's on-chain identity. The private key goes in `.env`.

---

### 6. Testnet ETH (Base Sepolia)
> **Used for:** gas fees when writing attestations to ERC-8004 registry

Get free testnet ETH from either:
- [Alchemy Base Sepolia Faucet](https://www.alchemy.com/faucets/base-sepolia) — paste your agent address
- [Coinbase Base Sepolia Faucet](https://faucet.quicknode.com/base/sepolia)

**0.01 ETH is more than enough** — Base Sepolia gas is nearly zero.

---

### 7. Hackathon Registration (Lablab)
> **Used for:** prize eligibility — required before anything else

1. Register on [early.surge.xyz](https://early.surge.xyz)
   - If prompted for credentials: `admin` / `JBRv2xWG7AzwVrLz88`
2. Register on [lablab.ai/ai-hackathons/ai-trading-agents](https://lablab.ai/ai-hackathons/ai-trading-agents)
3. Register on [Kite AI Hackathon](https://www.encodeclub.com/programmes/kites-hackathon-ai-agentic-economy)

---

## Step 1 — Create the GitHub Repository

```bash
# Create repo on GitHub first (public, MIT license, no template)
# Then locally:

git clone https://github.com/YOUR_USERNAME/arbmind.git
cd arbmind

# Create folder structure
mkdir -p src contracts scripts tests logs dashboard
touch src/__init__.py
touch .env .env.example .gitignore README.md
```

Add to `.gitignore`:
```
.env
logs/
__pycache__/
*.pyc
.venv/
node_modules/
artifacts/
cache/
out/
```

---

## Step 2 — Create `.env.example` and `.env`

`.env.example` (commit this):
```env
# Kraken Exchange
KRAKEN_API_KEY=your_kraken_api_key_here
KRAKEN_API_SECRET=your_kraken_api_secret_here

# PRISM API (Algo-Nomics)
PRISM_API_KEY=your_prism_api_key_here

# Anthropic (Claude strategy router)
ANTHROPIC_API_KEY=your_anthropic_api_key_here

# Web3 / Blockchain
WEB3_RPC_URL=https://base-sepolia.g.alchemy.com/v2/YOUR_ALCHEMY_KEY
AGENT_PRIVATE_KEY=0x_your_generated_private_key_here
AGENT_ADDRESS=0x_your_generated_address_here

# ERC-8004 Registry Contracts (Base Sepolia — already deployed, use as-is)
IDENTITY_REGISTRY_ADDR=0x8004A818BFB912233c491871b3d84c89A494BD9e
REPUTATION_REGISTRY_ADDR=0x8004B663056A597Dffe9eCcC1965A193B7388713

# Hackathon Capital Sandbox (get from lablab Discord after registering)
SANDBOX_VAULT_ADDR=
RISK_ROUTER_ADDR=

# Agent Config
PAPER_MODE=true          # Set false only when ready for live trading
MAX_POSITION_PCT=0.02    # 2% max per trade
MAX_DAILY_DRAWDOWN=0.05  # 5% daily loss limit
MIN_PROFIT_THRESHOLD=0.0015  # 0.15% minimum net spread
```

`.env` (never commit — copy from `.env.example` and fill in real values):
```bash
cp .env.example .env
# Edit .env with your real values
```

---

## Step 3 — Install Python Environment

```bash
# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# Install dependencies
pip install \
  python-dotenv \
  anthropic \
  web3 \
  eth-account \
  aiohttp \
  asyncio \
  requests
```

Create `requirements.txt`:
```txt
python-dotenv>=1.0.0
anthropic>=0.20.0
web3>=6.0.0
eth-account>=0.10.0
aiohttp>=3.9.0
requests>=2.31.0
```

---

## Step 4 — Install Kraken CLI (Official Binary)

> ⚠️ This is the official Kraken Rust binary — NOT the python-kraken-sdk.
> The Rust binary is what the hackathon uses.

```bash
# Install (macOS + Linux):
curl --proto '=https' --tlsv1.2 -LsSf \
  https://github.com/krakenfx/kraken-cli/releases/latest/download/kraken-cli-installer.sh | sh

# Verify install:
kraken --version

# Windows: use WSL2 and run the same command inside it
```

Configure with your API key (skip this if staying paper-only):
```bash
kraken auth set --key YOUR_KRAKEN_API_KEY --secret YOUR_KRAKEN_API_SECRET
```

---

## Step 5 — Verify Kraken CLI Paper Trading

Run these exactly as written — these are the **corrected commands** from the Kraken team:

```bash
# Initialize paper account with $10,000
kraken paper init --balance 10000 --currency USD

# Check paper balance
kraken paper balance

# Stream live BTC price (Ctrl+C to stop)
kraken ticker BTCUSD

# Same but as JSON (use -o json, NOT --json)
kraken -o json ticker BTCUSD

# Place a paper market buy
kraken paper buy BTCUSD 0.001

# Place a paper limit sell
kraken paper sell BTCUSD 0.001 --type limit --price 120000

# Check paper order status
kraken paper status

# View paper trade history
kraken paper history

# Reset paper account
kraken paper reset
```

> ❌ Commands that DO NOT exist (from the broken tutorial):
> - `kraken ticker --pair BTCUSD` → use `kraken ticker BTCUSD`
> - `kraken order add` → use `kraken order buy` or `kraken order sell`
> - `kraken --json` → use `kraken -o json`
> - `kraken mcp serve --port 8080` → MCP runs over stdio, not HTTP

> ✅ Correct live order commands (for when PAPER_MODE=false):
> ```bash
> kraken order buy BTCUSD 0.001 --type market
> kraken order buy BTCUSD 0.001 --type limit --price 95000
> kraken order sell BTCUSD 0.001 --type market
> kraken order cancel-after 60   # Dead man's switch — ALWAYS run this
> ```

---

## Step 6 — Verify PRISM API

```bash
# Quick test — replace YOUR_KEY with your actual PRISM key
curl "https://prism-api.algo-nomics.com/price/latest/?ticker=BTC&api_key=YOUR_KEY"

# Expected response:
# {"ticker": "BTC", "price": ..., "timestamp": ...}
```

Or in Python:
```python
import requests, os
from dotenv import load_dotenv
load_dotenv()

key = os.getenv("PRISM_API_KEY")
base = "https://prism-api.algo-nomics.com"

# Test all 4 endpoints we use
for endpoint in [
    f"{base}/price/latest/?ticker=BTC&api_key={key}",
    f"{base}/spotter/latest/?ticker=BTC&api_key={key}",
    f"{base}/sparc/latest/?ticker=BTC&api_key={key}",
    f"{base}/screener-ticker/?ticker=BTC&api_key={key}",
]:
    r = requests.get(endpoint)
    print(f"{endpoint.split('?')[0].split('/')[-2]}: {r.status_code} — {r.json()}")
```

---

## Step 7 — Verify Web3 + ERC-8004 Connection

```python
# scripts/verify_web3.py
from web3 import Web3
from eth_account import Account
import os
from dotenv import load_dotenv

load_dotenv()

# Connect to Base Sepolia
w3 = Web3(Web3.HTTPProvider(os.getenv("WEB3_RPC_URL")))
print(f"Connected: {w3.is_connected()}")
print(f"Chain ID: {w3.eth.chain_id}")   # Should be 84532 for Base Sepolia

# Verify agent wallet
private_key = os.getenv("AGENT_PRIVATE_KEY")
account = Account.from_key(private_key)
print(f"Agent address: {account.address}")

# Check testnet ETH balance
balance = w3.eth.get_balance(account.address)
print(f"ETH balance: {w3.from_wei(balance, 'ether')} ETH")

# Verify ERC-8004 Identity Registry is reachable
# Minimal ABI — just totalSupply to confirm contract exists
IDENTITY_ABI = [{"inputs":[],"name":"totalSupply","outputs":[{"type":"uint256"}],"stateMutability":"view","type":"function"}]
registry = w3.eth.contract(
    address=Web3.to_checksum_address(os.getenv("IDENTITY_REGISTRY_ADDR")),
    abi=IDENTITY_ABI
)
total_agents = registry.functions.totalSupply().call()
print(f"ERC-8004 Identity Registry reachable — {total_agents} agents registered")
```

Run it:
```bash
python scripts/verify_web3.py
```

Expected output:
```
Connected: True
Chain ID: 84532
Agent address: 0xYourAddress...
ETH balance: 0.05 ETH
ERC-8004 Identity Registry reachable — 1247 agents registered
```

---

## Step 8 — Project Structure Check

After completing all steps, your project should look like this:

```
arbmind/
├── .env                    ← your real secrets (never commit)
├── .env.example            ← template (commit this)
├── .gitignore
├── README.md
├── requirements.txt
├── src/
│   └── __init__.py
├── contracts/              ← Solidity contracts go here (Issue #7)
├── scripts/
│   └── verify_web3.py      ← what you just wrote
├── tests/
├── logs/
└── dashboard/
```

---

## Step 9 — Confirm All Credentials Work

Run this final checklist script:

```python
# scripts/healthcheck.py
import os, subprocess, requests
from web3 import Web3
from eth_account import Account
from dotenv import load_dotenv

load_dotenv()

print("=" * 50)
print("ARBMIND SETUP HEALTHCHECK")
print("=" * 50)

# 1. Kraken CLI
try:
    result = subprocess.run(["kraken", "--version"], capture_output=True, text=True)
    print(f"✅ Kraken CLI: {result.stdout.strip()}")
except FileNotFoundError:
    print("❌ Kraken CLI: not installed")

# 2. Kraken paper mode
try:
    result = subprocess.run(["kraken", "paper", "balance"], capture_output=True, text=True)
    if result.returncode == 0:
        print("✅ Kraken paper mode: working")
    else:
        print("⚠️  Kraken paper mode: needs init — run: kraken paper init --balance 10000")
except Exception as e:
    print(f"❌ Kraken paper: {e}")

# 3. PRISM API
try:
    key = os.getenv("PRISM_API_KEY")
    r = requests.get(f"https://prism-api.algo-nomics.com/price/latest/?ticker=BTC&api_key={key}", timeout=5)
    if r.status_code == 200:
        print(f"✅ PRISM API: connected, BTC = ${r.json().get('price', '?')}")
    else:
        print(f"❌ PRISM API: HTTP {r.status_code}")
except Exception as e:
    print(f"❌ PRISM API: {e}")

# 4. Anthropic
try:
    import anthropic
    client = anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))
    msg = client.messages.create(model="claude-sonnet-4-20250514", max_tokens=10, messages=[{"role":"user","content":"ping"}])
    print(f"✅ Anthropic API: connected")
except Exception as e:
    print(f"❌ Anthropic API: {e}")

# 5. Web3 + ERC-8004
try:
    w3 = Web3(Web3.HTTPProvider(os.getenv("WEB3_RPC_URL")))
    assert w3.is_connected(), "not connected"
    assert w3.eth.chain_id == 84532, f"wrong chain: {w3.eth.chain_id}"
    account = Account.from_key(os.getenv("AGENT_PRIVATE_KEY"))
    balance = w3.eth.get_balance(account.address)
    eth = float(w3.from_wei(balance, 'ether'))
    status = "✅" if eth > 0.001 else "⚠️ "
    print(f"{status} Web3 (Base Sepolia): connected, agent has {eth:.4f} ETH")
    print(f"   Agent address: {account.address}")
except Exception as e:
    print(f"❌ Web3: {e}")

# 6. ERC-8004 Registry
try:
    IDENTITY_ABI = [{"inputs":[],"name":"totalSupply","outputs":[{"type":"uint256"}],"stateMutability":"view","type":"function"}]
    registry = w3.eth.contract(
        address=Web3.to_checksum_address(os.getenv("IDENTITY_REGISTRY_ADDR")),
        abi=IDENTITY_ABI
    )
    total = registry.functions.totalSupply().call()
    print(f"✅ ERC-8004 Identity Registry: {total} agents registered on Base Sepolia")
except Exception as e:
    print(f"❌ ERC-8004 Registry: {e}")

print("=" * 50)
print("If all ✅ — you're ready for Issue #2")
print("=" * 50)
```

```bash
python scripts/healthcheck.py
```

---

## ERC-8004 Contracts — You Don't Need to Deploy Your Own

> This is the biggest time-saver for Issue #7: the official contracts are already live.

The `erc-8004/erc-8004-contracts` team maintains production-ready, audited contracts on multiple networks. **Use the pre-deployed ones** — don't redeploy unless you need custom logic.

**Base Sepolia (use these):**
| Contract | Address |
|---|---|
| IdentityRegistry | `0x8004A818BFB912233c491871b3d84c89A494BD9e` |
| ReputationRegistry | `0x8004B663056A597Dffe9eCcC1965A193B7388713` |

**Ethereum Sepolia (alternative):**
| Contract | Address |
|---|---|
| IdentityRegistry | `0x8004A169FB4a3325136EB29fA0ceB6D2e539a432` |
| ReputationRegistry | `0x8004BAa17C55a88189AE136b182e5fdA19dE9b63` |

Get the ABIs from the official repo:
```bash
git clone https://github.com/erc-8004/erc-8004-contracts.git
# ABIs are in: erc-8004-contracts/abis/
```

---

## Regarding the Tutorial Template

**Use the tutorial as a structural reference, but apply every correction below.**

The Kraken team flagged these specific bugs in the tutorial — they're fixed above but listed here for clarity:

| Tutorial says | Correct version |
|---|---|
| `KRAKEN_SANDBOX=true` in .env | Delete — doesn't exist |
| `kraken --sandbox` flag | Delete — doesn't exist |
| `kraken --json` | `kraken -o json` |
| `kraken ticker --pair BTCUSD` | `kraken ticker BTCUSD` |
| `kraken order add` | `kraken order buy` or `kraken order sell` |
| `kraken mcp serve --port 8080` | `kraken mcp` (stdio only) |
| `XBTUSD` as ticker | `BTCUSD` |
| "Rate-limit retries (no 429 handling needed)" | Rate-limit errors are surfaced immediately with `retryable`, `suggestion`, `docs_url` fields — your agent must handle them |

---

## What's Next (Issue #2)

Once your healthcheck shows all green:

1. `git add -A && git commit -m "feat: project setup, env config, healthcheck passing"`
2. `git push origin main`
3. Move to **Issue #2** — install and configure Kraken CLI Python wrapper + first paper trade loop

---

## Credentials Summary Card

Keep this somewhere safe:

```
Kraken API Key:     [from kraken.com → Security → API]
Kraken API Secret:  [same — only shown once]
PRISM API Key:      [from prism.algo-nomics.com → Settings]
Anthropic Key:      [from console.anthropic.com → API Keys]
RPC URL:            [from alchemy.com → Base Sepolia app]
Agent Private Key:  [generated with: cast wallet new]
Agent Address:      [printed by cast wallet new]
Base Sepolia ETH:   [from Alchemy faucet — paste agent address]

ERC-8004 IdentityRegistry  (Base Sepolia): 0x8004A818BFB912233c491871b3d84c89A494BD9e
ERC-8004 ReputationRegistry (Base Sepolia): 0x8004B663056A597Dffe9eCcC1965A193B7388713

Lablab early.surge.xyz:  admin / JBRv2xWG7AzwVrLz88
```
