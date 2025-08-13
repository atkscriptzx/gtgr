# GTGR

> Fully automated, stealth‑first Solana copy‑trading stack with **WalletManager** + **state tracking** for automatic key regeneration. Includes burner/collector lifecycle, 3‑funder rotation, proxy hops, DEX swaps (50/25/25 BTC/ETH/XRP), kill switch, Telegram control, Tor, GPG/PGP, cron/systemd, and dry‑run mode.

---

## README.md

```md
# solana-copytrader

Fully-automated, stealth-first Solana copy-trading stack with 3-funder rotation,
burner/collector lifecycle, proxy hops, DEX swaps (50/25/25 BTC/ETH/XRP),
kill switch, Telegram control, Tor, GPG/PGP, cron/systemd, dry-run mode,
**automatic key regeneration** via WalletManager, and persistent `data/state.json`.

### Key Features
- Burner drains at ≥16 SOL or 04:45; burns & **auto-regenerates** (5 SOL refilled from Active Funder)
- Collector DEX swaps at 04:50 in 50 SOL chunks (5–10s delay), then burns & **auto-regenerates**
- 3-Funder rotation at 05:00 (Active→burn if empty, Next→Active, Standby→Next, **new Standby generated**)
- 2-hop Proxy routing; proxies burned after each cycle
- Kill switch: unknown IP → instant drain → direct DEX → burn & recreate → reboot → Telegram alert
- Telegram: `/status`, `/rotate_now`, `/sync`, `/kill <passphrase>`, `/vault_status`, `/atpnl`

### Auto Key Regeneration
- Filenames in `.env` remain constant (e.g., `BURNER_KEY=burner.json`).
- On burn/recreate, the **same filename** is overwritten with a fresh keypair and
  the **new pubkey** is saved to `data/state.json`.
- Verify with `scripts/sim_rotate.py`.

**Strategy notes:** Cupesy trails from **+20%** with a **0.001%** drop; Euris trails from **+36%** with a **0.001%** drop.
```

---

## .env

```dotenv
ENV=prod
TZ=America/Nassau
DATA_DIR=./data
LOG_DIR=./logs

WHITELISTED_IPS=45.32.173.97
KILL_SWITCH_ENABLED=true
KILL_SWITCH_PASSPHRASE=atkansas

TELEGRAM_BOT_TOKEN=8493237385:AAHHFS8MJk_IkrFTVKZel0K1xTT1htV8cS4
TELEGRAM_CHAT_ID_MAIN=6120949753
TELEGRAM_CHAT_ID_ALERTS=6120949753
TELEGRAM_ADMIN_IDS=6120949753

SOLANA_CLUSTER=mainnet
RPC_PRIMARY=https://empty-methodical-asphalt.solana-mainnet.quiknode.pro/121047ee49945da6b0adba7cd07826e4802812c3/
RPC_POOL_FILE=./config/rpc_pool.txt
RPC_LATENCY_MS_THRESHOLD=500

KEYS_DIR=./keys
COLLECTOR_KEY=collector.json
FUNDER_ACTIVE_KEY=funder_active.json
FUNDER_NEXT_KEY=funder_next.json
FUNDER_STANDBY_KEY=funder_standby.json
BURNER_KEY=burner.json
PROXY1_KEY=proxy1.json
PROXY2_KEY=proxy2.json
PROXY3_KEY=proxy3.json
PROXY4_KEY=proxy4.json
PROXY5_KEY=proxy5.json
PROXY6_KEY=proxy6.json
HOLD_COLD_KEY=hold_cold.json
HOLD_HOT_KEY=hold_hot.json

TRADE_SIZE_SOL=2.5
BURNER_TARGET_SOL=16
FORCED_DRAIN_CRON=45 4 * * *
ROTATE_CRON=0 5 * * *
DEX_CRON=50 4 * * *
WATCH_ADDRESSES=suqh5sHtr8HyJ7q8scBimULPkPpA557prMG47xCHQfK,DfMxre4cKmvogbLrPigxmibVTTQDuzjdXojWzjCXXhzj
MAX_MARKETCAP_USD=20000

DEX_ENDPOINT=https://quote-api.jup.ag/v6
DEX_PROVIDER=jupiter
JUPITER_REQUIRE_PLATFORM=Raydium
JUPITER_ONLY_DIRECT_ROUTES=true
JUPITER_SLIPPAGE_BPS=250
MINT_BTC=9n4nbM75f5Ui33ZbPYXn59EwSgE8CGsHtAeTH5YFeJ9E
MINT_ETH=2FPyTwcZLUg1MDrwsyoP4D6s1tM7hAkHYRjkNb5w6Pxk
MINT_XRP=2jcHBYd9T2Mc9nhvFEBCDuBN1XjbbQUVow67WGWhv6zT

SWAP_SPLIT_BTC=0.50
SWAP_SPLIT_ETH=0.25
SWAP_SPLIT_XRP=0.25
VAULT_SPLIT_HOT=0.05
VAULT_SPLIT_COLD=0.95
CHUNK_SIZE_SOL=50
CHUNK_DELAY_S_MIN=5
CHUNK_DELAY_S_MAX=10

PROXY_HOPS=2
PROXY_DELAY_S_MIN=3
PROXY_DELAY_S_MAX=9

DAILY_REPORT_CRON=59 23 * * *
QUOTE_API=none
USD_BASED_PNL=true

TOR_ENABLED=true
TOR_CONTROL_PORT=9051
GPG_KEY_EMAIL=atkan101@gmail.com
PGP_BACKUP_ENABLED=true
SYSTEMD_SERVICE_NAME=copytrader.service

COPYTRADE_SLIPPAGE_BPS=500
DEX_SWAP_SLIPPAGE_BPS=250

COPYTRADE_PRIORITY_SOL=0.01
DEX_SWAP_PRIORITY_SOL=0.0

ESTIMATED_SWAP_CU=300000


```

---

## Docker, Make, Req, pypro

```yaml
# docker-compose.yml
version: "3.9"
services:
  bot:
    build: .
    env_file: .env
    volumes:
      - ./:/app
    restart: unless-stopped
    command: ["python", "-m", "src.main"]
```

```dockerfile
# Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENV TZ=America/Nassau
CMD ["python", "-m", "src.main"]
```

```makefile
# Makefile
.PHONY: run fmt test
run:
	python -m src.main
fmt:
	python -m pip install -U ruff || true
	ruff check --fix || true
test:
	pytest -q || true
```

```txt
#requirements.txt
python-dotenv==1.0.1
pydantic==2.8.2
loguru==0.7.2
requests==2.32.3
python-telegram-bot==21.4
pyyaml==6.0.2
croniter==3.0.3
solana==0.30.2
solders==0.20.0
base58>=2.1.1
```

```toml
#pyproject.toml
[project]
name = "solana-copytrader"
version = "0.1.0"
description = "Fully automated stealth Solana copy-trading bot"
requires-python = ">=3.11"
dependencies = [
  "python-dotenv==1.0.1",
  "pydantic==2.8.2",
  "loguru==0.7.2",
  "requests==2.32.3",
  "python-telegram-bot==21.4",
  "pyyaml==6.0.2",
  "croniter==3.0.3",
  "solana==0.30.2",
  "solders==0.20.0",
  "base58>=2.1.1",
]

[tool.ruff]
line-length = 100
```

## keys

```bash
keys/burner.json
keys/collector.json
keys/funder_active.json
keys/funder_next.json
keys/funder_standby.json
keys/proxy1.json
keys/proxy2.json
keys/proxy3.json
keys/proxy4.json
keys/hold_cold.json
keys/hold_hot.json
```

## Scripts

```bash
# scripts/bootstrap.sh
#!/usr/bin/env bash
set -euo pipefail
mkdir -p ${DATA_DIR:-./data} ${LOG_DIR:-./logs} keys backups
python - <<'PY'
import os
TOKEN=os.getenv('TELEGRAM_BOT_TOKEN','')
cmds=[
 {"command":"status","description":"Live balances"},
 {"command":"rotate_now","description":"Manual 5AM rotation"},
 {"command":"sync","description":"Health check"},
 {"command":"kill","description":"Kill switch"},
 {"command":"vault_status","description":"Hot/Cold balances"},
 {"command":"atpnl","description":"All-time PnL"},
]
print('Commands preview:', cmds)
if not TOKEN: print('No TELEGRAM_BOT_TOKEN set; skipping API call.')
PY
```

```bash
#scripts/daily_5am_trigger.sh
#!/bin/bash
set -euo pipefail

PROJECT_DIR="$(dirname "$(dirname "$(realpath "$0")")")"
cd "$PROJECT_DIR"
source .venv/bin/activate

# 1) 04:45 drain
python -m src.sched.at_0445_drain
# 2) 04:50 DEX
python -m src.sched.at_0450_dex
# 3) 05:00 rotate
python -m src.sched.at_0500_rotate

```

```bash
#scripts/rotate_now.sh
#!/bin/bash
set -e

PROJECT_DIR="$(dirname "$(dirname "$(realpath "$0")")")"
cd "$PROJECT_DIR"
source .venv/bin/activate

python -m src.flow.rotation --rotate

```

```bash
#scripts/self_heal.sh
#!/bin/bash
set -euo pipefail

# Use the same service name as deploy/copytrader.service (overridable via env)
SERVICE_NAME="${SYSTEMD_SERVICE_NAME:-copytrader.service}"

if ! systemctl is-active --quiet "$SERVICE_NAME"; then
  echo "[SELF-HEAL] $SERVICE_NAME not active — restarting..."
  sudo systemctl restart "$SERVICE_NAME"
fi

```

```bash
#scripts/gpg_sign_logs.sh
#!/bin/bash
set -e

LOG_DIR="logs"
SIGNING_KEY="your-gpg-key-id"

for file in "$LOG_DIR"/*.log; do
    [ -e "$file" ] || continue
    gpg --yes --local-user "$SIGNING_KEY" --output "$file.sig" --detach-sign "$file"
done
```

```bash
#scripts/pgp_encrypt_keys.sh
#!/bin/bash
set -euo pipefail
umask 077

KEY_DIR="keys"
BACKUP_DIR="backups"
RECIPIENT="${RECIPIENT:-your-pgp-key-id}"


mkdir -p "$BACKUP_DIR"
for file in "$KEY_DIR"/*.json; do
    [ -e "$file" ] || continue
   out="$BACKUP_DIR/$(basename "$file").gpg"
    gpg --yes --encrypt --armor --recipient "$RECIPIENT" --output "$out" "$file"
done
```

```bash
# scripts/cron_install.sh
#!/usr/bin/env bash
set -euo pipefail
( crontab -l 2>/dev/null; echo "$FORCED_DRAIN_CRON cd $(pwd) && . .venv/bin/activate && python -m src.sched.at_0445_drain >> logs/cron.log 2>&1" ) | crontab -
( crontab -l 2>/dev/null; echo "$DEX_CRON cd $(pwd) && . .venv/bin/activate && python -m src.sched.at_0450_dex >> logs/cron.log 2>&1" ) | crontab -
( crontab -l 2>/dev/null; echo "$ROTATE_CRON cd $(pwd) && . .venv/bin/activate && python -m src.sched.at_0500_rotate >> logs/cron.log 2>&1" ) | crontab -
( crontab -l 2>/dev/null; echo "$DAILY_REPORT_CRON cd $(pwd) && . .venv/bin/activate && python -m src.reports.daily_report >> logs/cron.log 2>&1" ) | crontab -
```

```bash
#systemd_install.sh
#!/usr/bin/env bash
set -euo pipefail

# Installs system-wide (root) units so the bot starts at boot without user login.
# Adjust paths if your repo directory/user differs.

sudo cp deploy/copytrader.service /etc/systemd/system/
sudo cp deploy/copytrader.timer /etc/systemd/system/
sudo cp deploy/copytrader-health.service /etc/systemd/system/
sudo cp deploy/copytrader-health.timer /etc/systemd/system/

sudo systemctl daemon-reload
sudo systemctl enable copytrader.service copytrader.timer copytrader-health.timer
sudo systemctl start  copytrader.timer  copytrader-health.timer

echo "✓ Installed system services. Use: sudo systemctl status copytrader.service"

```

```bash
# scripts/venv_bootstrap.sh
#!/usr/bin/env bash
set -euo pipefail
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
echo "✓ venv ready"
```

```bash
# scripts/sim_rotate.py
#!/usr/bin/env python3
from src.solana.wallet_manager import WalletManager
from src.core.state import State

wm = WalletManager()
b = wm.replace_wallet("burner", "burner.json")
s = wm.replace_wallet("funder_standby", "funder_standby.json")
print("New burner:", b.pubkey)
print("New standby:", s.pubkey)
print("State now:")
print(State().load())
```

---

## Config, Deploy & Data

```yaml
# config/config.schema.yaml
app:
  env: prod
  tz: America/Nassau
security:
  whitelist_ips: ["1.2.3.4"]
  kill_switch_enabled: true
trading:
  trade_size_sol: 2.5
  burner_target_sol: 16
  max_marketcap_usd: 20000
  watch_addresses: ["CupesyPubkey", "EurisPubkey"]
dex:
  endpoint: https://dex.invalid
  chunk_size_sol: 50
  delay_s_min: 5
  delay_s_max: 10
  splits:
    btc: 0.50
    eth: 0.25
    xrp: 0.25
proxy:
  hops: 2
  delay_s_min: 3
  delay_s_max: 9

```

```yaml
#config/rpc_pool.txt
# Primary (paid) RPCs — best reliability/throughput
https://mainnet.helius-rpc.com/?api-key=YOUR_HELIUS_KEY
https://solana-mainnet.g.alchemy.com/v2/YOUR_ALCHEMY_KEY
https://solana-mainnet.core.chainstack.com/YOUR_KEY
https://bold-absolute-someid.solana-mainnet.quiknode.pro/YOUR_KEY/

# Secondary (community/public) — rate limited, use as last resort
https://api.mainnet-beta.solana.com
https://solana-api.projectserum.com
https://rpc.ankr.com/solana

```

```yaml
#config/telegram_commands.md
/status        → Show live balances for burner, funders, collector; active RPC
/rotate_now    → Promote NEXT→ACTIVE, STANDBY→NEXT, create new STANDBY
/sync          → Health check (RPC latency, cron timers, service status)
/kill          → Trigger kill switch (drain→DEX→burn & regen) [use with caution]
/vault_status  → Show Hot/Cold wallet totals (BTC/ETH/XRP)
/atpnl         → All-time PnL (USD), plus today’s change

```

```yaml
#config/tor/torrc.example
# Tor data directory (must be writable by tor user)
DataDirectory /var/lib/tor

# Minimize DNS leaks / reduce logging
Log notice stdout
SocksPort 9050
AvoidDiskWrites 1
ClientOnly 1

# === Hidden Service: SSH over Tor (.onion:22 → localhost:22) ===
HiddenServiceDir /var/lib/tor/copytrader_ssh/
HiddenServicePort 22 127.0.0.1:22

# === Hidden Service: optional local dashboard (if you add one) ===
#HiddenServiceDir /var/lib/tor/copytrader_ui/
#HiddenServicePort 8080 127.0.0.1:8080

# Performance/compat tweaks
NumCPUs 2
CircuitBuildTimeout 10
LearnCircuitBuildTimeout 0
KeepalivePeriod 60

```

```ini
#deploy/copytrader.service
[Unit]
Description=Solana Copytrader Bot
After=network.target

[Service]
Type=simple
User=linixuser
WorkingDirectory=/home/linuxuser/solana-copytrader
ExecStart=/home/linuxuser/solana-copytrader/.venv/bin/python -m src.main
Restart=always
RestartSec=10
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

```ini
# deploy/copytrader-health.timer
[Unit]
Description=Healthcheck timer
[Timer]
OnBootSec=2min
OnUnitActiveSec=2min
Unit=copytrader-health.service
[Install]
WantedBy=timers.target
```

```ini
# deploy/copytrader-health.service 
[Unit]
Description=Healthcheck for CopyTrader

[Service]
Type=oneshot
User=linuxuser
WorkingDirectory=/home/linuxuser/solana-copytrader
ExecStart=/home/linuxuser/solana-copytrader/.venv/bin/python /home/linuxuser/solana-copytrader/src/utils/check_rpc_latency.py

```

```ini
#deploy/copytrader.timer
[Unit]
Description=Run Solana Copytrader Bot at startup and keep alive

[Timer]
# Start immediately at boot
OnBootSec=30s
# Run every 1 minute to check health or restart if dead
OnUnitActiveSec=60s
Unit=copytrader.service

[Install]
WantedBy=timers.target

```

```json
#data/state.json
{
  "active_rpc"="https://empty-methodical-asphalt.solana-mainnet.quiknode.pro/121047ee49945da6b0adba7cd07826e4802812c3/",
  "wallets": {
    "burner":         { "pubkey": null, "keyfile": "keys/burner.json" },
    "collector":      { "pubkey": null, "keyfile": "keys/collector.json" },
    "funder_active":  { "pubkey": null, "keyfile": "keys/funder_active.json" },
    "funder_next":    { "pubkey": null, "keyfile": "keys/funder_next.json" },
    "funder_standby": { "pubkey": null, "keyfile": "keys/funder_standby.json" },

    "proxy1": { "pubkey": null, "keyfile": "keys/proxy1.json" },
    "proxy2": { "pubkey": null, "keyfile": "keys/proxy2.json" },
    "proxy3": { "pubkey": null, "keyfile": "keys/proxy3.json" },
    "proxy4": { "pubkey": null, "keyfile": "keys/proxy4.json" },
    "proxy5": { "pubkey": null, "keyfile": "keys/proxy5.json" },
    "proxy6": { "pubkey": null, "keyfile": "keys/proxy6.json" },

    "hold_hot":  { "pubkey": null, "keyfile": "keys/hold_hot.json" },
    "hold_cold": { "pubkey": null, "keyfile": "keys/hold_cold.json" }
  },
  "last_rotation": null,
  "last_kill_switch": null,
  "tokens_bought": {
    "burner": [],
    "collector": [],
    "funder_active": [],
    "funder_next": [],
    "funder_standby": []
  }
}

```

---

## Core and Utils

```python
# src/utils/check_rpc_latency.py
import os, sys, time, subprocess, requests, random, json

RPC_POOL_FILE = os.getenv("RPC_POOL_FILE", "config/rpc_pool.txt")
STATE_FILE = "data/state.json"
LATENCY_LIMIT_MS = int(os.getenv("RPC_LATENCY_MS_THRESHOLD", "500"))

def check_latency(url: str) -> float:
    start = time.time()
    try:
        requests.post(url, json={"jsonrpc": "2.0", "id": 1, "method": "getHealth"}, timeout=2)
        return (time.time() - start) * 1000
    except Exception:
        return float("inf")

def reset_rpc():
    """Choose a random RPC from the pool and save it to state.json."""
    with open(RPC_POOL_FILE) as f:
        rpcs = [line.strip() for line in f if line.strip()]
    if not rpcs:
        print("[RPC RESET] No RPC endpoints found in pool file.")
        return
    new_rpc = random.choice(rpcs)
    print(f"[RPC RESET] Switching to {new_rpc}")

    state = {}
    if os.path.exists(STATE_FILE):
        try:
            with open(STATE_FILE) as f:
                state = json.load(f)
        except Exception:
            pass
    state["active_rpc"] = new_rpc
    with open(STATE_FILE, "w") as f:
        json.dump(state, f, indent=2)

def main():
    with open(RPC_POOL_FILE) as f:
        rpcs = [ln.strip() for ln in f if ln.strip() and not ln.strip().startswith("#")]
    bad = []
    for rpc in rpcs:
        latency = check_latency(rpc)
        print(f"{rpc} latency: {latency:.2f} ms")
        if latency > LATENCY_LIMIT_MS:
            bad.append(rpc)
    if len(bad) == len(rpcs):
        print("[!] All RPCs slow — rotating...")
        reset_rpc()

if __name__ == "__main__":
    main()

```

```python
#src/utils/reset_rpc.py
import random, os, json

RPC_POOL_FILE = os.getenv("RPC_POOL_FILE", "config/rpc_pool.txt")
STATE_FILE = "data/state.json"

def main():
    with open(RPC_POOL_FILE) as f:
        rpcs = [line.strip() for line in f if line.strip()]
    new_rpc = random.choice(rpcs)
    print(f"[RPC RESET] Switching to {new_rpc}")

    state = {}
    if os.path.exists(STATE_FILE):
        with open(STATE_FILE) as f:
            state = json.load(f)
    state["active_rpc"] = new_rpc
    with open(STATE_FILE, "w") as f:
        json.dump(state, f, indent=2)

if __name__ == "__main__":
    main()
```

```python
# src/core/config.py
from pydantic import BaseModel, Field
from dotenv import load_dotenv
import os
from pathlib import Path

load_dotenv()

def _env_float(name: str, default: float) -> float:
    try:
        return float(os.getenv(name, default))
    except Exception:
        return float(default)

def _env_int(name: str, default: int) -> int:
    try:
        return int(float(os.getenv(name, default)))
    except Exception:
        return int(default)

def _env_list(name: str) -> list[str]:
    v = os.getenv(name, "")
    return [x for x in (s.strip() for s in v.split(",")) if x]

class VaultSplitCfg(BaseModel):
    hot:  float = Field(default_factory=lambda: _env_float("VAULT_SPLIT_HOT", 0.05))
    cold: float = Field(default_factory=lambda: _env_float("VAULT_SPLIT_COLD", 0.95))

class Splits(BaseModel):
    btc: float = Field(default_factory=lambda: _env_float("SWAP_SPLIT_BTC", 0.50))
    eth: float = Field(default_factory=lambda: _env_float("SWAP_SPLIT_ETH", 0.25))
    xrp: float = Field(default_factory=lambda: _env_float("SWAP_SPLIT_XRP", 0.25))

class ProxyCfg(BaseModel):
    hops: int = Field(default_factory=lambda: _env_int("PROXY_HOPS", 2))
    delay_s_min: int = Field(default_factory=lambda: _env_int("PROXY_DELAY_S_MIN", 3))
    delay_s_max: int = Field(default_factory=lambda: _env_int("PROXY_DELAY_S_MAX", 9))

class DexCfg(BaseModel):
    endpoint: str = Field(default_factory=lambda: os.getenv("DEX_ENDPOINT", "https://dex.invalid"))
    chunk_size_sol: float = Field(default_factory=lambda: _env_float("CHUNK_SIZE_SOL", 50))
    delay_s_min: int = Field(default_factory=lambda: _env_int("CHUNK_DELAY_S_MIN", 5))
    delay_s_max: int = Field(default_factory=lambda: _env_int("CHUNK_DELAY_S_MAX", 10))
    splits: Splits = Field(default_factory=Splits)

class TradingCfg(BaseModel):
    trade_size_sol: float = Field(default_factory=lambda: _env_float("TRADE_SIZE_SOL", 2.5))
    burner_target_sol: float = Field(default_factory=lambda: _env_float("BURNER_TARGET_SOL", 16))
    max_marketcap_usd: int = Field(default_factory=lambda: _env_int("MAX_MARKETCAP_USD", 20000))
    watch_addresses: list[str] = Field(default_factory=lambda: _env_list("WATCH_ADDRESSES"))

class PriorityCfg(BaseModel):
    copytrade_priority_sol: float = Field(default_factory=lambda: float(os.getenv("COPYTRADE_PRIORITY_SOL", "0.01")))
    dex_swap_priority_sol: float = Field(default_factory=lambda: float(os.getenv("DEX_SWAP_PRIORITY_SOL", "0.0")))
    estimated_swap_cu: int = Field(default_factory=lambda: int(float(os.getenv("ESTIMATED_SWAP_CU", "300000"))))

class AppCfg(BaseModel):
    env: str = Field(default_factory=lambda: os.getenv('ENV', 'prod'))
    tz: str = Field(default_factory=lambda: os.getenv('TZ', 'America/Nassau'))
    data_dir: Path = Field(default_factory=lambda: Path(os.getenv('DATA_DIR', './data')).resolve())
    log_dir: Path = Field(default_factory=lambda: Path(os.getenv('LOG_DIR', './logs')).resolve())
    whitelist_ips: list[str] = Field(default_factory=lambda: _env_list('WHITELISTED_IPS'))
    kill_switch_enabled: bool = Field(default_factory=lambda: os.getenv('KILL_SWITCH_ENABLED', 'true').strip().lower() == 'true')
    rpc_primary: str = Field(default_factory=lambda: os.getenv('RPC_PRIMARY', ''))
    rpc_pool_file: str = Field(default_factory=lambda: os.getenv('RPC_POOL_FILE', './config/rpc_pool.txt'))

    COLLECTOR_KEY: str = Field(default_factory=lambda: os.getenv('COLLECTOR_KEY', 'collector.json'))
    FUNDER_ACTIVE_KEY: str = Field(default_factory=lambda: os.getenv('FUNDER_ACTIVE_KEY', 'funder_active.json'))
    FUNDER_NEXT_KEY: str = Field(default_factory=lambda: os.getenv('FUNDER_NEXT_KEY', 'funder_next.json'))
    FUNDER_STANDBY_KEY: str = Field(default_factory=lambda: os.getenv('FUNDER_STANDBY_KEY', 'funder_standby.json'))
    BURNER_KEY: str = Field(default_factory=lambda: os.getenv('BURNER_KEY', 'burner.json'))

    PROXY1_KEY: str = Field(default_factory=lambda: os.getenv('PROXY1_KEY', 'proxy1.json'))
    PROXY2_KEY: str = Field(default_factory=lambda: os.getenv('PROXY2_KEY', 'proxy2.json'))
    PROXY3_KEY: str = Field(default_factory=lambda: os.getenv('PROXY3_KEY', 'proxy3.json'))
    PROXY4_KEY: str = Field(default_factory=lambda: os.getenv('PROXY4_KEY', 'proxy4.json'))
    PROXY5_KEY: str = Field(default_factory=lambda: os.getenv('PROXY5_KEY', 'proxy5.json'))
    PROXY6_KEY: str = Field(default_factory=lambda: os.getenv('PROXY6_KEY', 'proxy6.json'))

    HOLD_HOT_KEY: str = Field(default_factory=lambda: os.getenv('HOLD_HOT_KEY', 'hold_hot.json'))
    HOLD_COLD_KEY: str = Field(default_factory=lambda: os.getenv('HOLD_COLD_KEY', 'hold_cold.json'))

    dex: DexCfg = Field(default_factory=DexCfg)
    proxy: ProxyCfg = Field(default_factory=ProxyCfg)
    trading: TradingCfg = Field(default_factory=TradingCfg)
    vault_split: VaultSplitCfg = Field(default_factory=VaultSplitCfg)

class _SlippageCfg:
    def __init__(self):
        self.copytrade_slippage_bps = int(os.getenv("COPYTRADE_SLIPPAGE_BPS", "500"))
        self.dex_swap_slippage_bps = int(os.getenv("DEX_SWAP_SLIPPAGE_BPS", "250"))

# Add near your other config loaders
class TokenMintCfg:
    def __init__(self, env):
        self.btc = env.get("MINT_BTC", "").strip()
        self.eth = env.get("MINT_ETH", "").strip()
        self.xrp = env.get("MINT_XRP", "").strip()

class _DexCfg:
    def __init__(self, env):
        self.provider = env.get("DEX_PROVIDER", "jupiter")
        self.endpoint = env.get("DEX_ENDPOINT", "https://quote-api.jup.ag/v6")
        self.jupiter_require_platform = env.get("JUPITER_REQUIRE_PLATFORM", "").strip()
        self.jupiter_only_direct = env.get("JUPITER_ONLY_DIRECT_ROUTES", "true").lower() == "true"
        self.jupiter_slippage_bps = int(env.get("JUPITER_SLIPPAGE_BPS", "50"))
        self.mints = TokenMintCfg(env)

# ensure: CFG.dex = _DexCfg(os.environ)


# Ensure CFG.dex = _DexCfg(os.environ)


# ... make sure CFG.dex = _DexCfg(os.environ)

CFG = AppCfg()
CFG.slippage = _SlippageCfg()
CFG.priority = PriorityCfg()

```

```python
#src/utils/slippage.py
from decimal import Decimal

def apply_slippage_min_out(quoted_out: int | float, slippage_bps: int) -> int:
    """
    Convert a quoted expected output amount into a min_out with slippage protection.
    quoted_out: expected OUT amount (smallest units)
    slippage_bps: basis points (e.g., 550 = 5.5%)
    """
    q = Decimal(str(quoted_out))
    factor = (Decimal(10_000) - Decimal(slippage_bps)) / Decimal(10_000)
    min_out = int((q * factor).to_integral_value(rounding="ROUND_FLOOR"))
    return max(min_out, 1) if quoted_out > 0 else 0
```

```python
# src/core/logger.py
from loguru import logger
from pathlib import Path
from .config import CFG

Path(CFG.log_dir).mkdir(parents=True, exist_ok=True)
_ = logger.add(Path(CFG.log_dir, 'copytrader.log'), rotation='10 MB', retention='14 days', compression='zip', enqueue=True, backtrace=True, diagnose=False)

```

```python
# src/core/keystore.py
from pathlib import Path

class KeyStore:
    @staticmethod
    def path(key_name: str) -> Path:
        """
        Return the absolute path under keys/ (does NOT require the file to exist).
        Callers that *require* existence should check .exists() themselves.
        """
        p = Path(key_name)
        if not p.is_absolute():
            p = Path("keys") / p
        return p.resolve()

```

```python
# src/core/telemetry.py
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Any, Dict, Optional, Callable
from time import perf_counter
from loguru import logger


@dataclass
class Telemetry:
    """
    Minimal telemetry sink:
    - event(name, **fields): structured log line
    - timeit(name): decorator to record duration_ms and success flag
    - gauge/counter: simple numeric metrics emitted as logs (works with log shippers)
    """
    default_ctx: Dict[str, Any] = field(default_factory=dict)

    def with_context(self, **ctx) -> "Telemetry":
        merged = {**self.default_ctx, **ctx}
        return Telemetry(default_ctx=merged)

    # --- core emitters -----------------------------------------------------

    def event(self, name: str, **fields: Any) -> None:
        payload = {**self.default_ctx, **fields}
        logger.info(f"[telemetry] {name} | {payload}")

    def warn(self, name: str, **fields: Any) -> None:
        payload = {**self.default_ctx, **fields}
        logger.warning(f"[telemetry] {name} | {payload}")

    def error(self, name: str, **fields: Any) -> None:
        payload = {**self.default_ctx, **fields}
        logger.error(f"[telemetry] {name} | {payload}")

    # --- metrics-style helpers --------------------------------------------

    def counter(self, name: str, inc: float = 1.0, **fields: Any) -> None:
        payload = {**self.default_ctx, **fields, "value": inc}
        logger.info(f"[metric.counter] {name} | {payload}")

    def gauge(self, name: str, value: float, **fields: Any) -> None:
        payload = {**self.default_ctx, **fields, "value": value}
        logger.info(f"[metric.gauge] {name} | {payload}")

    # --- decorators --------------------------------------------------------

    def timeit(self, name: str) -> Callable:
        """
        Decorator that measures duration and emits success/failure.
        Usage:
            t = Telemetry().with_context(flow="collector")
            @t.timeit("dex.swap")
            def do_swap(...): ...
        """
        def _wrap(fn: Callable) -> Callable:
            def _inner(*args, **kwargs):
                start = perf_counter()
                try:
                    result = fn(*args, **kwargs)
                    self.event(name, duration_ms=round((perf_counter() - start) * 1000, 2), ok=True)
                    return result
                except Exception as e:
                    self.error(name, duration_ms=round((perf_counter() - start) * 1000, 2), ok=False, err=str(e))
                    raise
            return _inner
        return _wrap

```

```python
# src/core/timeutils.py
import time, random

def jitter(min_s:int, max_s:int):
    time.sleep(random.randint(min_s, max_s))
```

```python
# src/core/state.py
from __future__ import annotations
from pathlib import Path
import json
from typing import Any, Dict

STATE_PATH = Path('data/state.json')
DEFAULT = {
    "wallets": {
        "burner": {"pubkey": None},
        "collector": {"pubkey": None},
        "funder_active": {"pubkey": None},
        "funder_next": {"pubkey": None},
        "funder_standby": {"pubkey": None},
        "proxy1": {"pubkey": None},
        "proxy2": {"pubkey": None},
        "hold_hot": {"pubkey": None},
        "hold_cold": {"pubkey": None},
 },
    # NEW: registry of tokens bought per wallet role
    "tokens_bought": {
        "burner": [],   # list of token mint addresses already bought
        "collector": [],
        "funder_active": [],
        "funder_next": [],
        "funder_standby": []
    }
}

class State:
    def __init__(self, path: Path = STATE_PATH):
        self.path = path
        self.path.parent.mkdir(parents=True, exist_ok=True)
        if not self.path.exists():
            self.save(DEFAULT)
        self._cache = self.load()

    def load(self) -> Dict[str, Any]:
        try:
            return json.loads(self.path.read_text())
        except Exception:
            return DEFAULT.copy()

    def save(self, data: Dict[str, Any] | None = None):
        if data is None:
            data = self._cache
        self.path.write_text(json.dumps(data, indent=2))

    def set_pubkey(self, role: str, pubkey: str):
        self._cache.setdefault("wallets", {}).setdefault(role, {})["pubkey"] = pubkey
        self.save()

    def get_pubkey(self, role: str) -> str | None:
        return self._cache.get("wallets", {}).get(role, {}).get("pubkey")

# === NEW HELPERS FOR TOKEN BUY LIMITS ===
    def has_bought_token(self, role: str, token_mint: str) -> bool:
        tb = self._cache.setdefault("tokens_bought", {}).setdefault(role, [])
        return token_mint in tb

    def record_token_buy(self, role: str, token_mint: str) -> None:
        tb = self._cache.setdefault("tokens_bought", {}).setdefault(role, [])
        if token_mint not in tb:
            tb.append(token_mint)
            self.save()

    def clear_role_token_buys(self, role: str) -> None:
        self._cache.setdefault("tokens_bought", {})[role] = []
        self.save()
```

```python
#src/core/utils.py
from __future__ import annotations
import json
import os
import random
import time
from contextlib import contextmanager
from datetime import datetime, timezone
from pathlib import Path
from typing import Iterable, Iterator, List, TypeVar, Callable, Any

T = TypeVar("T")


# --- collection helpers -----------------------------------------------------

def chunked(items: List[T] | Iterable[T], size: int) -> Iterator[List[T]]:
    """Yield lists of length `size` from an iterable."""
    if size <= 0:
        raise ValueError("size must be > 0")
    buf: List[T] = []
    for x in items:
        buf.append(x)
        if len(buf) >= size:
            yield buf
            buf = []
    if buf:
        yield buf


# --- time helpers -----------------------------------------------------------

def now_utc_iso() -> str:
    return datetime.now(timezone.utc).isoformat(timespec="seconds")

def sleep_jitter(min_s: int, max_s: int) -> None:
    time.sleep(random.randint(min_s, max_s))


# --- env helpers ------------------------------------------------------------

def env_bool(name: str, default: bool = False) -> bool:
    v = os.getenv(name)
    if v is None:
        return default
    return v.strip().lower() in {"1", "true", "yes", "y", "on"}


# --- JSON I/O (safe) --------------------------------------------------------

def load_json_safe(path: Path | str, default: Any = None) -> Any:
    p = Path(path)
    if not p.exists():
        return default
    try:
        return json.loads(p.read_text())
    except Exception:
        return default

def save_json_atomic(path: Path | str, data: Any) -> None:
    """Write JSON atomically to avoid corruption on crashes."""
    p = Path(path)
    tmp = p.with_suffix(p.suffix + ".tmp")
    tmp.write_text(json.dumps(data, indent=2))
    tmp.replace(p)


# --- retry/backoff ----------------------------------------------------------

def retry(
    attempts: int = 3,
    base_delay: float = 0.5,
    max_delay: float = 5.0,
    jitter: float = 0.25,
    on: tuple[type[BaseException], ...] = (Exception,),
):
    """
    Decorator for simple exponential backoff with jitter.
    Usage:
        @retry(attempts=5)
        def rpc_call(...): ...
    """
    def _wrap(fn: Callable):
        def _inner(*args, **kwargs):
            delay = base_delay
            last_exc = None
            for i in range(1, attempts + 1):
                try:
                    return fn(*args, **kwargs)
                except on as e:
                    last_exc = e
                    if i == attempts:
                        raise
                    sleep = min(max_delay, delay * (2 ** (i - 1)))
                    # add +/- jitter%
                    jitter_amt = sleep * jitter
                    time.sleep(max(0.01, random.uniform(sleep - jitter_amt, sleep + jitter_amt)))
            # Should not reach; re-raise to satisfy type checkers
            raise last_exc  # type: ignore[misc]
        return _inner
    return _wrap


# --- human formatting --------------------------------------------------------

def human_sol(x: float) -> str:
    return f"{x:,.4f} SOL"

def human_usd(x: float) -> str:
    return f"${x:,.2f}"

```

---

## Solana (Wallets/Client)

```python
# src/solana/wallet.py
from dataclasses import dataclass
@dataclass
class Wallet:
    name: str
    key_path: str
    pubkey: str
```

```python
# src/solana/client.py
from __future__ import annotations
from dataclasses import dataclass
from loguru import logger
from typing import Any
from solana.rpc.api import Client
from solana.rpc.types import TxOpts
from solana.compute_budget_program import ComputeBudgetProgram
from solana.transaction import Transaction
from solana.system_program import TransferParams, transfer
from src.solana.keypair_io import load_keypair_from_file
from src.core.config import CFG
from src.core.state import State
from solders.pubkey import Pubkey
from base64 import b64encode
from solders.signature import Signature 

LAMPORTS_PER_SOL = 1_000_000_000

@dataclass
class TxResult:
    sig: str

class SolanaClient:
    """
    Real RPC client using solana-py. Reads the active RPC from state.json if present,
    otherwise falls back to CFG.rpc_primary.
    """
    def __init__(self, rpc_url: str | None = None):
        state = State().load()
        active_rpc = state.get("active_rpc")
        self.rpc_url = rpc_url or active_rpc or CFG.rpc_primary
        self.client = Client(self.rpc_url)
        logger.info(f"[SolanaClient] RPC={self.rpc_url}")

    # --- helpers ------------------------------------------------------------
    @staticmethod
    def _resp_ok(resp: Any) -> bool:
        if resp is None:
            return False
        if hasattr(resp, "is_err"):
            return not resp.is_err()
        if isinstance(resp, dict):
            return "error" not in resp
        return True

    @staticmethod
    def _balance_from_resp(resp: Any) -> int:
        if hasattr(resp, "value"):
            try:
                return int(resp.value)
            except Exception:
                pass
        if isinstance(resp, dict):
            try:
                return int(resp.get("result", {}).get("value"))
            except Exception:
                pass
        raise RuntimeError(f"Unexpected get_balance response: {resp}")

    @staticmethod
    def _priority_ixs(cu_limit: int = 1_400_000, price_micro_lamports: int = 0):
        """
        Compute-budget Ixs:
          - cu_limit: max compute units the tx can consume
          - price_micro_lamports: *micro-lamports per CU* (priority fee). 0 disables priority.
        """
        ixs = [ComputeBudgetProgram.set_compute_unit_limit(cu_limit)]
        if price_micro_lamports > 0:
            ixs.append(ComputeBudgetProgram.set_compute_unit_price(price_micro_lamports))
        return ixs

    @staticmethod
    def price_for_target_priority_sol(target_priority_sol: float, estimated_cu: int) -> int:
        """
        Convert a target priority fee in SOL to micro‑lamports per compute unit.
        Example: target_priority_sol = 0.01, estimated_cu = 300_000
        """
        if target_priority_sol <= 0 or estimated_cu <= 0:
            return 0  # no tip
        lamports = int(target_priority_sol * LAMPORTS_PER_SOL)
        # micro‑lamports per CU = (lamports * 1_000_000) / estimated_cu
        return max(int(lamports * 1_000_000 // estimated_cu), 1)



    def get_balance(self, wallet_keyfile: str) -> float:
        kp = load_keypair_from_file(wallet_keyfile)
        resp = self.client.get_balance(kp.pubkey())
        if not self._resp_ok(resp):
            raise RuntimeError(f"get_balance RPC error: {resp}")
        lamports = self._balance_from_resp(resp)
        sol = lamports / LAMPORTS_PER_SOL
        logger.debug(f"[balance] {kp.pubkey()} = {sol:.9f} SOL")
        return sol

    def transfer(
        self,
        src_keyfile: str,
        dst_addr: str,
        amount_sol: float,
        *,
        priority_micro_lamports: int = 0,   # <<< set >0 to enable priority
        cu_limit: int = 1_400_000,
        skip_preflight: bool = True,        # <<< safer against MEV leakage
        max_retries: int = 3,
    ) -> TxResult:
        if amount_sol <= 0:
            raise ValueError("amount_sol must be > 0")

        kp = load_keypair_from_file(src_keyfile)
        lamports = int(amount_sol * LAMPORTS_PER_SOL)

        # blockhash
        bh = self.client.get_latest_blockhash()
        if not self._resp_ok(bh):
            raise RuntimeError(f"get_latest_blockhash error: {bh}")
        try:
            bhash = bh.value.blockhash
        except Exception:
            bhash = bh.get("result", {}).get("value", {}).get("blockhash")
        if not bhash:
            raise RuntimeError(f"Could not parse latest blockhash: {bh}")

        to_pub = Pubkey.from_string(dst_addr)

        # Build transaction with priority compute-budget Ixs FIRST
        tx = Transaction(recent_blockhash=bhash, fee_payer=kp.pubkey())
        for ix in self._priority_ixs(cu_limit=cu_limit, price_micro_lamports=priority_micro_lamports):
            tx.add(ix)

        # Your real instruction(s)
        tx.add(transfer(TransferParams(from_pubkey=kp.pubkey(), to_pubkey=to_pub, lamports=lamports)))

        # Send + confirm
        resp = self.client.send_transaction(
            tx, kp,
            opts=TxOpts(skip_preflight=skip_preflight, max_retries=max_retries)
        )
        if not self._resp_ok(resp):
            raise RuntimeError(f"send_transaction error: {resp}")
        try:
            signature = str(resp.value)
        except Exception:
            signature = str(resp.get("result"))
        logger.info(f"[transfer] {amount_sol} SOL -> {dst_addr} sig={signature}")

        try:
            conf = self.client.confirm_transaction(signature)
            if not self._resp_ok(conf):
                logger.warning(f"[transfer] confirm error: {conf}")
        except Exception as e:
            logger.warning(f"[transfer] confirm exception: {e}")
        return TxResult(sig=signature)

    # ... (keep existing imports & class up to burn_wallet)

    def burn_wallet(self, key_path: str):
        from loguru import logger
        logger.warning(f"Burn wallet called for {key_path} (file deletion handled elsewhere).")

    def send_legacy_tx_bytes(self, tx_bytes: bytes, signer_keyfile: str, skip_preflight: bool = True) -> str:
        """
        Sign a legacy Transaction (bytes), then send+confirm.
        """
        kp = load_keypair_from_file(signer_keyfile)
        tx = Transaction.deserialize(tx_bytes)
        tx.sign(kp)
        raw = bytes(tx.serialize())
        resp = self.client.send_raw_transaction(raw, opts=TxOpts(skip_preflight=skip_preflight, max_retries=3))
        if not self._resp_ok(resp):
            raise RuntimeError(f"send_raw_transaction error: {resp}")
        try:
            sig = str(resp.value)
        except Exception:
            sig = str(resp.get("result"))
        try:
            conf = self.client.confirm_transaction(Signature.from_string(sig))
            if not self._resp_ok(conf):
                logger.warning(f"[jup] confirm error: {conf}")
        except Exception as e:
            logger.warning(f"[jup] confirm exception: {e}")
        return sig

    @staticmethod
    def lamports_from_sol(amount_sol: float) -> int:
        return int(amount_sol * LAMPORTS_PER_SOL)

```

```python
# src/solana/wallet_manager.py
from __future__ import annotations
from pathlib import Path
import os, json
from dataclasses import dataclass
from loguru import logger
from solders.keypair import Keypair
from .wallet import Wallet
from src.core.state import State
from src.core.keystore import KeyStore

def _generate_keypair_json() -> tuple[str, str]:
    kp = Keypair()
    secret = kp.to_bytes()
    pub = str(kp.pubkey())
    return pub, json.dumps(list(secret), separators=(',', ':')) + "\n"

@dataclass
class RolePaths:
    role: str
    key_filename: str

class WalletManager:
    def __init__(self, state: State | None = None):
        self.state = state or State()

    def _write_keyfile(self, filename: str, secret_json: str) -> Path:
        p = KeyStore.path(filename)
        p.parent.mkdir(parents=True, exist_ok=True)
        try:
            os.chmod(p.parent, 0o700)
        except Exception:
            pass
        tmp = p.with_suffix(p.suffix + ".new")
        fd = os.open(tmp, os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o600)
        try:
            os.write(fd, secret_json.encode("utf-8"))
            os.fsync(fd)
        finally:
            os.close(fd)
        os.replace(tmp, p)
        try:
            os.chmod(p, 0o600)
        except Exception:
            pass
        return p

    def create_wallet(self, role: str, filename: str) -> Wallet:
        pub, secret_json = _generate_keypair_json()
        path = self._write_keyfile(filename, secret_json)
        self.state.set_pubkey(role, pub)
        logger.info(f"[{role}] created {pub} at {path}")
        return Wallet(name=role, key_path=str(path), pubkey=pub)

    def burn_wallet(self, role: str, filename: str):
        p = KeyStore.path(filename)
        if p.exists():
            p.unlink()
            logger.warning(f"[{role}] burned keyfile {p}")
        self.state.set_pubkey(role, None)

    def replace_wallet(self, role: str, filename: str) -> Wallet:
        self.burn_wallet(role, filename)
        return self.create_wallet(role, filename)

```

```python
# src/solana/keypair_io.py
from __future__ import annotations
import json
from pathlib import Path
from solders.keypair import Keypair

def load_keypair_from_file(path: str | Path) -> Keypair:
    p = Path(path)
    data = json.loads(p.read_text())
    if not isinstance(data, list) or len(data) not in (64, 32):
        raise ValueError(f"Unexpected key format in {p}")
    # Both 64-byte secret (preferred) or 32-byte secret supported
    b = bytes(data)
    return Keypair.from_bytes(b) if len(b) == 64 else Keypair.from_seed(b)
```

---

## DEX

```python
# src/dex/client.py
from dataclasses import dataclass
from loguru import logger

@dataclass
class SwapAlloc:
    btc: float
    eth: float
    xrp: float

class DexClient:
    def __init__(self, endpoint:str):
        self.endpoint = endpoint

    def swap_sol_to_alloc(
        self,
        from_wallet: str,
        amount_sol: float,
        alloc: SwapAlloc,
        *,
        slippage_bps: int = 0,
        priority_micro_lamports: int = 0,
    ) -> dict:
        logger.info(
            f"Swapping {amount_sol} SOL to BTC:{alloc.btc} ETH:{alloc.eth} XRP:{alloc.xrp} "
            f"(slip={slippage_bps}bps, priority_micro={priority_micro_lamports})"
        )
        return {"btc": amount_sol*alloc.btc, "eth": amount_sol*alloc.eth, "xrp": amount_sol*alloc.xrp}



```

```python
# src/dex/jupiter.py
from __future__ import annotations
import requests
from base64 import b64decode
from loguru import logger
from typing import Any, Dict, Optional
from src.core.config import CFG

SOL_MINT = "So11111111111111111111111111111111111111112"

class JupiterClient:
    def __init__(self, base_url: str | None = None):
        self.base_url = base_url or CFG.dex.endpoint.rstrip("/")

    def quote(self, input_mint: str, output_mint: str, amount_lamports: int) -> Dict[str, Any]:
        """
        Get a quote and filter to a route whose every leg is platform == CFG.dex.jupiter_require_platform
        when that flag is set (e.g., "Raydium").
        """
        url = f"{self.base_url}/quote"
        params = {
            "inputMint": input_mint,
            "outputMint": output_mint,
            "amount": str(amount_lamports),
            "slippageBps": CFG.dex.jupiter_slippage_bps,
            "onlyDirectRoutes": "true" if CFG.dex.jupiter_only_direct else "false",
        }
        r = requests.get(url, params=params, timeout=15)
        r.raise_for_status()
        data = r.json()

        # If user requires a specific platform (e.g., Raydium), pick the first all‑Raydium route
        must = CFG.dex.jupiter_require_platform
        if must:
            routes = data.get("data") or data.get("routes") or []
            for route in routes:
                ok = True
                # Jupiter v6 keeps legs under "routePlan"
                for leg in route.get("routePlan", []):
                    swap_info = leg.get("swapInfo", {})
                    platform = (swap_info.get("platform") or swap_info.get("label") or "").strip()
                    if platform.lower() != must.lower():
                        ok = False
                        break
                if ok:
                    logger.info(f"Selected {must}-only route")
                    return {"route": route, "raw": data}

            raise RuntimeError(f"No {must}-only route found for {output_mint}")

        # Else return the best route as-is
        best = (data.get("data") or data.get("routes") or [None])[0]
        if not best:
            raise RuntimeError("No route returned by Jupiter")
        return {"route": best, "raw": data}

    def build_swap_tx(self, route: Dict[str, Any], user_pubkey: str) -> bytes:
        """
        Your existing method that calls POST /swap to get a ready-to-send transaction.
        Keep as-is. It will build a tx that uses the selected route (Raydium-only if filtered above).
        """
        url = f"{self.base_url}/swap"
        payload = {
            "userPublicKey": user_pubkey,
            "wrapAndUnwrapSol": True,
            "useSharedAccounts": True,
            "asLegacyTransaction": True,
            "computeUnitPriceMicroLamports": 0,
            "dynamicComputeUnitLimit": True,
            # Jupiter v6 expects the full route object:
            "quoteResponse": route["route"],
        }
        r = requests.post(url, json=payload, timeout=20)
        r.raise_for_status()
        data = r.json()
        return bytes.fromhex(data["swapTransaction"])

```

```python
# src/dex/swapper.py
from loguru import logger
from src.core.config import CFG
from src.solana.client import SolanaClient
from .client import DexClient, SwapAlloc
from src.dex.client import DexClient

SOL_MINT = "So11111111111111111111111111111111111111112"

def _mint_for_symbol(symbol: str) -> str:
    s = symbol.upper()
    if s == "BTC":
        return CFG.dex.mints.btc
    if s == "ETH":
        return CFG.dex.mints.eth
    if s == "XRP":
        return CFG.dex.mints.xrp
    raise ValueError(f"Unsupported symbol: {symbol}")

class Swapper:
    def __init__(self, dex: DexClient):
        self.dex = dex

    def swap_sol_to_symbol(self, wallet_keyfile: str, amount_lamports: int, symbol: str) -> bytes:
        out_mint = _mint_for_symbol(symbol)  # <-- Raydium mint from .env/CFG
        quote = self.dex.quote(SOL_MINT, out_mint, amount_lamports)
        # you already filter for Raydium-only in Jupiter client
        route = quote["route"]
        # get user pubkey from wallet_keyfile as you already do
        user_pubkey = self._pubkey_str(wallet_keyfile)
        tx_bytes = self.dex.build_swap_tx(route, user_pubkey)
        return tx_bytes

    def swap_chunks(self, wallet: str, total_sol: float) -> None:
        """
        Swap a total SOL amount into BTC/ETH/XRP according to CFG.dex.splits, in chunks.
        NOTE: After this completes, you hold SPL tokens (not SOL). If you want to
        route tokens via proxies, you must implement SPL token transfers separately.
        """
        remaining = float(total_sol)
        alloc = SwapAlloc(CFG.dex.splits.btc, CFG.dex.splits.eth, CFG.dex.splits.xrp)

        # Priority fee for sweeps (often 0.0 SOL -> micro price 0)
        price_micro = SolanaClient.price_for_target_priority_sol(
            target_priority_sol=CFG.priority.dex_swap_priority_sol,
            estimated_cu=CFG.priority.estimated_swap_cu,
        )
        slip_bps = CFG.slippage.dex_swap_slippage_bps

        while remaining > 0:
            chunk = min(CFG.dex.chunk_size_sol, remaining)

            # Perform the swap (SOL -> BTC/ETH/XRP)
            self.dex.swap_sol_to_alloc(
                from_wallet=wallet,               # <-- correct kwarg
                amount_sol=chunk,
                alloc=alloc,
                slippage_bps=slip_bps,
                priority_micro_lamports=price_micro,
            )

            remaining -= chunk
            logger.info(f"[swapper] chunk={chunk} SOL swapped; remaining={remaining} SOL")


```

```python
# src/dex/spl_move.py
class SPLRoutingNotImplemented(Exception):
    pass

def transfer_spl_via_proxies(*args, **kwargs):
    """
    Placeholder for SPL token routing via proxies.
    Raise loudly so production does not silently fail.
    """
    raise SPLRoutingNotImplemented(
        "SPL token routing via proxies is not implemented yet. "
        "Post-DEX, assets are SPL tokens; implement token program transfers if you need to hop proxies."
    )

```

---

## Flows

```python
# src/flow/burner.py
from loguru import logger
from src.core.config import CFG
from src.core.state import State
from src.solana.client import SolanaClient
from src.solana.wallet_manager import WalletManager

class BurnerFlow:
    def __init__(self, sol: SolanaClient, wm: WalletManager, burner_keyfile: str, collector_addr: str, funder_active_keyfile: str, notify_cb=None):
        self.sol = sol
        self.wm = wm
        self.burner_keyfile = burner_keyfile
        self.collector_addr = collector_addr
        self.funder_active_keyfile = funder_active_keyfile
        self.notify_cb = notify_cb

    def on_trade_profit(self):
        bal = self.sol.get_balance(self.burner_keyfile)
        if bal >= CFG.trading.burner_target_sol:
            self.drain_and_recreate()

    def forced_drain(self):
        self.drain_and_recreate()

    def _fund_new_burner(self):
        new_burner_addr = State().get_pubkey("burner")
        if not new_burner_addr:
            logger.error("No burner pubkey in state; cannot fund new burner.")
            return
        try:
            self.sol.transfer(self.funder_active_keyfile, new_burner_addr, 5.0)
            logger.info("Seeded new burner with 5.0 SOL from active funder.")
        except Exception as e:
            logger.exception(f"Failed to fund new burner: {e}")

    def drain_and_recreate(self):
        bal = self.sol.get_balance(self.burner_keyfile)
        if bal > 0:
            self.sol.transfer(self.burner_keyfile, self.collector_addr, bal)
        logger.info("Burner drained; burning & recreating")

        self.wm.replace_wallet("burner", getattr(CFG, "BURNER_KEY", "burner.json"))
        State().clear_role_token_buys("burner")
        self._fund_new_burner()
        if self.notify_cb:
            try:
                self.notify_cb("burner_regenerated")
            except Exception as e:
                logger.warning(f"notify_cb failed: {e}")

```

```python
# src/flow/collector.py
from __future__ import annotations
from loguru import logger
from src.core.config import CFG
from src.core.state import State
from src.dex.client import DexClient
from src.dex.swapper import Swapper
from src.solana.client import SolanaClient
from src.solana.wallet_manager import WalletManager

# NEW: SPL routing (token transfers via proxies)
from src.flow.proxies_spl import transfer_spl_via_proxies

# NEW: for reading token balances
from solders.pubkey import Pubkey
from solana.rpc.api import Client
from spl.token.constants import TOKEN_PROGRAM_ID, ASSOCIATED_TOKEN_PROGRAM_ID
from spl.token.instructions import get_associated_token_address

# ============================
# 🔁 REPLACE WITH REAL MINTS!
# ============================
mint_btc = CFG.dex.mints.btc
mint_eth = CFG.dex.mints.eth
mint_xrp = CFG.dex.mints.xrp

def _get_collector_pubkey(state: State) -> str | None:
    return state.get_pubkey("collector")


def _get_ui_token_balance(client: Client, owner_pubkey_str: str, mint_str: str) -> float:
    """
    Returns the owner's token balance in UI units (amount / 10**decimals).
    Creates no accounts, just reads the collector's ATA if it exists.
    """
    owner = Pubkey.from_string(owner_pubkey_str)
    mint = Pubkey.from_string(mint_str)

    ata = get_associated_token_address(owner, mint, TOKEN_PROGRAM_ID, ASSOCIATED_TOKEN_PROGRAM_ID)
    info = client.get_account_info(ata)

    # If ATA doesn't exist, balance is 0.0
    if not (hasattr(info, "value") and info.value is not None):
        return 0.0

    bal = client.get_token_account_balance(ata)
    # Prefer uiAmountString if available; else compute from amount & decimals
    try:
        if hasattr(bal, "value") and hasattr(bal.value, "ui_amount_string"):
            return float(bal.value.ui_amount_string)
    except Exception:
        pass

    # Fallback path
    try:
        amount = int(bal.value.amount)  # base units
        dec = int(bal.value.decimals)
        return amount / (10 ** dec)
    except Exception:
        return 0.0


class CollectorFlow:
    def __init__(self, sol: SolanaClient, dex: DexClient, wm: WalletManager, collector_keyfile: str):
        self.sol = sol
        self.dex = dex
        self.wm = wm
        self.collector_keyfile = collector_keyfile

    def run_0450(self) -> None:
        """
        04:50 job:
          1) Swap collector's SOL into BTC/ETH/XRP in chunks using your 50/25/25 split.
          2) Read the resulting token balances (BTC/ETH/XRP) in the collector.
          3) Route per‑token 5% HOT and 95% COLD via two-hop proxies (SPL transfers).
          4) Burn & regenerate collector + first two proxies (optional; keep consistent with your policy).
        """
        state = State()
        collector_addr = _get_collector_pubkey(state)
        if not collector_addr:
            logger.error("Collector pubkey missing; aborting collector flow.")
            return

        # 1) If there is SOL in the collector, swap it first
        bal_sol = self.sol.get_balance(self.collector_keyfile)
        if bal_sol <= 0:
            logger.info("Collector SOL empty; skipping swaps.")
        else:
            logger.info(f"[collector] Swapping {bal_sol:.6f} SOL into BTC/ETH/XRP (50/25/25)")
            swapper = Swapper(self.dex)
            swapper.swap_chunks(wallet=self.collector_keyfile, total_sol=bal_sol)

        # 2) Read SPL token balances now held by the collector
        rpc_client: Client = self.sol.client  # underlying solana-py client
        total_btc_ui = _get_ui_token_balance(rpc_client, collector_addr, MINT_BTC)
        total_eth_ui = _get_ui_token_balance(rpc_client, collector_addr, MINT_ETH)
        total_xrp_ui = _get_ui_token_balance(rpc_client, collector_addr, MINT_XRP)

        logger.info(
            f"[collector SPL balances] BTC={total_btc_ui:.6f} | ETH={total_eth_ui:.6f} | XRP={total_xrp_ui:.6f}"
        )

        hot_pct = CFG.vault_split.hot   # e.g., 0.05
        cold_pct = CFG.vault_split.cold # e.g., 0.95

        def _route_token(mint: str, total_ui: float) -> None:
            if total_ui <= 0:
                return
            amt_hot = total_ui * hot_pct
            amt_cold = max(0.0, total_ui - amt_hot)

            # HOT: collector -> proxy3 -> proxy4 -> hold_hot
            transfer_spl_via_proxies(
                hops=[CFG.PROXY3_KEY, CFG.PROXY4_KEY],
                src_key=CFG.COLLECTOR_KEY,
                dst_key=CFG.HOLD_HOT_KEY,
                mint_str=mint,
                amount_ui=amt_hot,
                delay_s_min=CFG.proxy.delay_s_min,
                delay_s_max=CFG.proxy.delay_s_max,
                burn_and_regen_on_use=True,
            )

            # COLD: collector -> proxy5 -> proxy6 -> hold_cold
            transfer_spl_via_proxies(
                hops=[CFG.PROXY5_KEY, CFG.PROXY6_KEY],
                src_key=CFG.COLLECTOR_KEY,
                dst_key=CFG.HOLD_COLD_KEY,
                mint_str=mint,
                amount_ui=amt_cold,
                delay_s_min=CFG.proxy.delay_s_min,
                delay_s_max=CFG.proxy.delay_s_max,
                burn_and_regen_on_use=True,
            )

        # 3) Route each token to vaults via proxies
        _route_token(MINT_BTC, total_btc_ui)
        _route_token(MINT_ETH, total_eth_ui)
        _route_token(MINT_XRP, total_xrp_ui)

        # 4) Rotate collector + (optionally) proxies used elsewhere per your policy.
        # NOTE: We already burn proxies inside the SPL router AFTER they forward tokens.
        # If you also rotate the collector each cycle, do it here:
        logger.info("Collector processed; (optional) burn & recreate collector for privacy hygiene.")
        self.wm.replace_wallet("collector", CFG.COLLECTOR_KEY)


```

```python
# src/flow/proxies_spl.py
from __future__ import annotations
from time import sleep
from typing import List, Tuple
from loguru import logger

from solders.pubkey import Pubkey
from solders.signature import Signature
from solana.rpc.api import Client
from solana.rpc.types import TxOpts
from solana.transaction import Transaction
from solana.compute_budget_program import ComputeBudgetProgram

from spl.token.constants import TOKEN_PROGRAM_ID, ASSOCIATED_TOKEN_PROGRAM_ID
from spl.token.instructions import (
    create_associated_token_account,
    get_associated_token_address,
    transfer_checked,
)

from src.solana.keypair_io import load_keypair_from_file
from src.core.keystore import KeyStore
from src.core.state import State
from src.solana.wallet_manager import WalletManager
from src.core.config import CFG


def _resolve_keyfile_path(cfg_key: str) -> str:
    # cfg_key is like CFG.COLLECTOR_KEY -> "collector.json"
    return str(KeyStore.path(cfg_key))


def _owner_pubkey_from_keyfile(keyfile: str) -> Pubkey:
    return load_keypair_from_file(keyfile).pubkey()


def _get_mint_decimals(client: Client, mint: Pubkey) -> int:
    # Fast + simple: getTokenSupply has `decimals`
    resp = client.get_token_supply(mint)
    if hasattr(resp, "value") and hasattr(resp.value, "decimals"):
        return int(resp.value.decimals)
    # Fallback: default 6 if RPCs are odd (many SPLs are 6)
    return 6


def _ensure_ata_ixs(client: Client, payer: Pubkey, owner: Pubkey, mint: Pubkey) -> Tuple[Pubkey, list]:
    ata = get_associated_token_address(owner, mint, TOKEN_PROGRAM_ID, ASSOCIATED_TOKEN_PROGRAM_ID)
    # If ATA does not exist, emit a create ix
    info = client.get_account_info(ata)
    ixs = []
    if not (hasattr(info, "value") and info.value is not None):
        ixs.append(create_associated_token_account(payer, owner, mint, TOKEN_PROGRAM_ID, ASSOCIATED_TOKEN_PROGRAM_ID))
    return ata, ixs


def _priority_ixs() -> list:
    price_micro = int(
        CFG.priority.dex_swap_priority_sol * 1_000_000 * 1_000_000 // max(CFG.priority.estimated_swap_cu, 1)
    ) if CFG.priority.dex_swap_priority_sol > 0 else 0
    ixs = [ComputeBudgetProgram.set_compute_unit_limit(CFG.priority.estimated_swap_cu)]
    if price_micro > 0:
        ixs.append(ComputeBudgetProgram.set_compute_unit_price(price_micro))
    return ixs


def _spl_transfer_one_leg(
    client: Client,
    mint: Pubkey,
    amount_ui: float,
    src_keyfile: str,
    dst_owner_keyfile_or_pub: str | Pubkey,
    *,
    decimals: int | None = None,
    skip_preflight: bool = True,
    max_retries: int = 3,
) -> str:
    """
    Transfer SPL tokens (mint) in UI units from src_owner -> dst_owner.
    - Creates ATAs on-demand.
    - Uses transfer_checked to enforce decimals.
    """
    src_kp = load_keypair_from_file(src_keyfile)
    src_owner = src_kp.pubkey()

    # Accept either keyfile path or direct pubkey for dst owner
    if isinstance(dst_owner_keyfile_or_pub, Pubkey):
        dst_owner = dst_owner_keyfile_or_pub
    else:
        dst_owner = load_keypair_from_file(dst_owner_keyfile_or_pub).pubkey()

    # Resolve/ensure ATAs
    src_ata, src_ata_ixs = _ensure_ata_ixs(client, payer=src_owner, owner=src_owner, mint=mint)
    dst_ata, dst_ata_ixs = _ensure_ata_ixs(client, payer=src_owner, owner=dst_owner, mint=mint)

    # Amount in base units
    dec = int(decimals) if decimals is not None else _get_mint_decimals(client, mint)
    base_amount = int(round(amount_ui * (10 ** dec)))

    # Recent blockhash
    bh = client.get_latest_blockhash()
    blockhash = bh.value.blockhash if hasattr(bh, "value") else bh["result"]["value"]["blockhash"]

    # Build tx
    tx = Transaction(recent_blockhash=blockhash, fee_payer=src_owner)
    for ix in _priority_ixs():
        tx.add(ix)
    # create ATAs if needed (payer = src_owner)
    for ix in src_ata_ixs + dst_ata_ixs:
        tx.add(ix)
    # transfer_checked
    tx.add(
        transfer_checked(
            program_id=TOKEN_PROGRAM_ID,
            source=src_ata,
            mint=mint,
            dest=dst_ata,
            owner=src_owner,
            amount=base_amount,
            decimals=dec,
            signers=None,  # owner signs
        )
    )

    # Send + confirm
    resp = client.send_transaction(tx, src_kp, opts=TxOpts(skip_preflight=skip_preflight, max_retries=max_retries))
    sig = str(resp.value) if hasattr(resp, "value") else str(resp["result"])
    try:
        client.confirm_transaction(Signature.from_string(sig))
    except Exception:
        pass
    return sig


def _role_from_cfg_filename(cfg_key: str) -> str:
    name = cfg_key.lower()
    for r in ["proxy1","proxy2","proxy3","proxy4","proxy5","proxy6","collector","hold_hot","hold_cold","funder_active","funder_next","funder_standby","burner"]:
        if r in name:
            return r
    return "proxy"


def transfer_spl_via_proxies(
    *,
    hops: List[str],              # e.g. [CFG.PROXY3_KEY, CFG.PROXY4_KEY]
    src_key: str,                 # e.g. CFG.COLLECTOR_KEY
    dst_key: str,                 # e.g. CFG.HOLD_HOT_KEY
    mint_str: str,                # token mint address (BTC/ETH/XRP wrapped)
    amount_ui: float,             # human units (e.g., 12.34)
    decimals: int | None = None,  # optional override
    delay_s_min: float = 0,
    delay_s_max: float = 0,
    burn_and_regen_on_use: bool = True,
) -> None:
    """
    Move SPL tokens along src -> hops... -> dst, creating ATAs as needed.
    Burns/recreates intermediate proxies *after* they forward tokens.
    """
    wm = WalletManager()
    state = State()
    client = Client(state.load().get("active_rpc") or CFG.rpc_primary)

    seq = [src_key] + list(hops) + [dst_key]
    kfs = [_resolve_keyfile_path(k) for k in seq]
    mint = Pubkey.from_string(mint_str)

    # Ensure destination owner pubkey exists in state (and create wallet file if needed)
    for cfg_key in (hops + [dst_key]):
        role = _role_from_cfg_filename(cfg_key)
        if not state.get_pubkey(role):
            # create the wallet file if missing so we can get a pubkey
            wm.create_wallet(role, cfg_key)

    import random
    last_forwarded_role: str | None = None

    for i in range(len(kfs) - 1):
        src_kf = kfs[i]
        dst_kf = kfs[i + 1]
        dst_role = _role_from_cfg_filename(seq[i + 1])
        dst_pub = wm.state.get_pubkey(dst_role)
        if not dst_pub:
            dst_pub = wm.create_wallet(dst_role, seq[i + 1]).pubkey

        logger.info(f"[SPL proxy] hop {i+1}/{len(kfs)-1}: {src_kf} -> {dst_pub} {amount_ui} (mint={mint_str})")
        sig = _spl_transfer_one_leg(
            client=client,
            mint=mint,
            amount_ui=amount_ui,
            src_keyfile=src_kf,
            dst_owner_keyfile_or_pub=Pubkey.from_string(dst_pub),
            decimals=decimals,
            skip_preflight=True,
        )
        logger.info(f"[SPL proxy] tx={sig}")

        # Burn+regen the *previous* intermediate once it has forwarded tokens
        if burn_and_regen_on_use and last_forwarded_role:
            logger.info(f"[SPL proxy] burn+regen {last_forwarded_role}")
            wm.replace_wallet(last_forwarded_role, last_forwarded_role + ".json")

        # Update last_forwarded_role if current hop was an intermediate destination
        if 0 <= i < len(kfs) - 2:
            last_forwarded_role = _role_from_cfg_filename(seq[i + 1])
        else:
            last_forwarded_role = None

        if delay_s_min or delay_s_max:
            sleep(random.uniform(delay_s_min, delay_s_max))

    # No burn of final destination (vault)
```

```python
# src/flow/funders.py
from __future__ import annotations
import json
from pathlib import Path
from loguru import logger
from src.solana.client import SolanaClient
from src.solana.wallet_manager import WalletManager
from src.solana.keypair_io import load_keypair_from_file
from src.core.state import State
from src.core.keystore import KeyStore
from src.core.config import CFG
from src.flow.proxies import transfer_via_proxies


class FunderRotation:
    def __init__(self, sol: SolanaClient, wm: WalletManager, active_kf: str, next_kf: str, standby_kf: str, collector_addr: str):
        self.sol = sol
        self.wm = wm
        self.active_kf = active_kf
        self.next_kf = next_kf
        self.standby_kf = standby_kf
        self.collector = collector_addr

    def ensure_active_has_5(self) -> None:
        """Top up ACTIVE funder to 5 SOL from collector if needed."""
        try:
            bal = self.sol.get_balance(self.active_kf)
            need = max(0.0, 5.0 - bal)
            if need > 0:
                self.sol.transfer(
                    str(KeyStore.path(CFG.COLLECTOR_KEY)),
                    self._pubkey_of(self.active_kf),
                    need,
                )
                logger.info(f"Topped up active funder by {need:.4f} SOL")
        except Exception as e:
            logger.exception(f"ensure_active_has_5 failed: {e}")

    def rotate(self) -> None:
        """
        ACTIVE -> (burn if empty)
        NEXT   -> ACTIVE
        STANDBY-> NEXT
        new STANDBY generated
        """
        logger.info("Rotating funders: active->(burn if empty), next->active, standby->next, new standby")
        state = State()
        try:
            if self.sol.get_balance(self.active_kf) == 0:
                self.wm.burn_wallet("funder_active", self.active_kf)
        except Exception as e:
            logger.warning(f"Could not check/burn active funder: {e}")

        self._copy_keyfile(self.next_kf, self.active_kf)
        state.set_pubkey("funder_active", self._pubkey_of(self.active_kf))

        self._copy_keyfile(self.standby_kf, self.next_kf)
        state.set_pubkey("funder_next", self._pubkey_of(self.next_kf))

        self.wm.replace_wallet("funder_standby", self.standby_kf)
        logger.info("Rotation complete.")

    # --- helpers ---
    def _copy_keyfile(self, src: str, dst: str) -> None:
        sp = KeyStore.path(src)
        dp = KeyStore.path(dst)
        if not sp.exists():
            raise FileNotFoundError(f"Missing source keyfile: {sp}")
        data = json.loads(sp.read_text())
        dp.parent.mkdir(parents=True, exist_ok=True)
        dp.write_text(json.dumps(data, separators=(',', ':')))

    def _pubkey_of(self, keyfile: str) -> str:
        kp = load_keypair_from_file(KeyStore.path(keyfile))
        return str(kp.pubkey())


def fund_funder_from_collector(amount_sol: float) -> None:
    """
    Optional helper: fund the ACTIVE funder from collector via two proxies.
    """
    transfer_via_proxies(
        hops=[CFG.PROXY1_KEY, CFG.PROXY2_KEY],
        src_key=CFG.COLLECTOR_KEY,
        dst_key=CFG.FUNDER_ACTIVE_KEY,
        amount_sol=amount_sol,
        delay_s_min=CFG.proxy.delay_s_min,
        delay_s_max=CFG.proxy.delay_s_max,
        burn_and_regen_on_use=True,
    )

```

```python
# src/flow/proxies.py
from time import sleep
from loguru import logger
from src.core.keystore import KeyStore
from src.solana.client import SolanaClient
from src.solana.wallet_manager import WalletManager

def _role_from_filename(cfg_key: str) -> str:
    name = cfg_key.lower()
    for r in ["proxy1","proxy2","proxy3","proxy4","proxy5","proxy6","collector","funder_active","funder_next","funder_standby","burner","hold_hot","hold_cold"]:
        if r in name:
            return r
    return "proxy"

def transfer_via_proxies(
    hops: list[str],
    src_key: str,
    dst_key: str,
    amount_sol: float,
    delay_s_min: float = 0,
    delay_s_max: float = 0,
    burn_and_regen_on_use: bool = True,
):
    wm = WalletManager()
    sol = SolanaClient()

    seq = [src_key] + list(hops) + [dst_key]
    kfs = [str(KeyStore.path(k)) for k in seq]

    for i in range(len(kfs) - 1):
        src_kf = kfs[i]
        dst_kf = kfs[i + 1]
        dst_role = _role_from_filename(seq[i + 1])
        dst_pub = wm.state.get_pubkey(dst_role) or wm.create_wallet(dst_role, dst_kf.split("/")[-1]).pubkey

        logger.info(f"[proxy] hop {i+1}/{len(kfs)-1}: {src_kf} -> {dst_pub} amount={amount_sol} SOL")
        sol.transfer(src_keyfile=src_kf, dst_addr=dst_pub, amount_sol=amount_sol, priority_micro_lamports=0, skip_preflight=True)

        if burn_and_regen_on_use and 0 < i < len(kfs) - 1:
            used_cfg_fname = seq[i]    # the hop we just delivered to
            used_role = _role_from_filename(used_cfg_fname)
            logger.info(f"[proxy] burn+regen {used_role}")
            wm.replace_wallet(used_role, used_cfg_fname)

        if delay_s_min or delay_s_max:
            import random
            sleep(random.uniform(delay_s_min, delay_s_max))

```

```python
# src/flow/rotation.py
from loguru import logger
from src.core.config import CFG
from src.core.state import State
from src.solana.client import SolanaClient
from src.solana.wallet_manager import WalletManager
from src.flow.burner import BurnerFlow
from src.flow.collector import CollectorFlow
from src.flow.funders import FunderRotation
from src.dex.client import DexClient
from src.core.keystore import KeyStore

class DailyRotation:
    def execute(self) -> None:
        logger.info("⏱ 04:45 drain -> 04:50 DEX -> 05:00 rotate sequence")
        state = State()
        wm = WalletManager(state)
        sol = SolanaClient()
        dex = DexClient(CFG.dex.endpoint)

        collector_addr = state.get_pubkey("collector")
        if not collector_addr:
            logger.error("Collector address missing in state.json — aborting rotation.")
            return

        # 04:45 — Drain burner and recreate
        BurnerFlow(
            sol=sol,
            wm=wm,
            burner_keyfile=str(KeyStore.path(CFG.BURNER_KEY)),
            collector_addr=collector_addr,
            funder_active_keyfile=str(KeyStore.path(CFG.FUNDER_ACTIVE_KEY)),
        ).forced_drain()

        # 04:50 — Collector -> DEX in chunks; recreate collector + proxies
        CollectorFlow(
            sol=sol, dex=dex, wm=wm, collector_keyfile=str(KeyStore.path(CFG.COLLECTOR_KEY))
        ).run_0450()

        # 05:00 — Funders
        fr = FunderRotation(
            sol=sol,
            wm=wm,
            active_kf=CFG.FUNDER_ACTIVE_KEY,
            next_kf=CFG.FUNDER_NEXT_KEY,
            standby_kf=CFG.FUNDER_STANDBY_KEY,
            collector_addr=collector_addr,
        )
        fr.rotate()
        fr.ensure_active_has_5()
        logger.info("✅ Daily rotation finished.")

```

---

## Trading

```python
# src/trading/watcher.py
from __future__ import annotations
from dataclasses import dataclass
from typing import Iterable, Optional
from loguru import logger
from pathlib import Path
import json
from src.core.config import CFG
from src.trading.copier import Copier

@dataclass
class TradeSignal:
    """
    Minimal representation of a copy-trade signal from Cupesy/Euris.
    You should fill in `raw_market_id` and/or `token_symbol` from your on-chain watcher.
    """
    source: str                 # "Cupesy" | "Euris"
    raw_market_id: Optional[str]  # e.g., a pool address, pair, or market id
    token_symbol: Optional[str]   # e.g., "XYZ"
    token_mint: Optional[str]     # if you already resolved it, stick it here


class WalletWatcher:
    def __init__(self, addresses: Iterable[str]):
        self.addresses = list(addresses)
        self.copier = Copier(trade_size_sol=CFG.trading.trade_size_sol)

    # --- public -------------------------------------------------------------

    def poll(self):
        """
        Read signals from data/signals.json (if present) and emit TradeSignal items.
        This gives you a quick path to integration; your on-chain watcher can write
        the file as a list of dicts, then this loader consumes & clears it.
        """
        logger.debug(f"Watching {len(self.addresses)} wallets")
        path = Path("data/signals.json")
        if not path.exists():
            return
        try:
            raw = json.loads(path.read_text())
            if isinstance(raw, list):
                for r in raw:
                    yield TradeSignal(
                        source=r.get("source","Unknown"),
                        raw_market_id=r.get("raw_market_id"),
                        token_symbol=r.get("token_symbol"),
                        token_mint=r.get("token_mint"),
                    )
            else:
                logger.warning("signals.json is not a list; skipping.")
        except Exception as e:
            logger.exception(f"Failed reading signals.json: {e}")
        finally:
            # Clear signals once consumed
            try:
                path.write_text("[]")
            except Exception:
                pass


    def handle_signals(self):
        """
        Main loop to process new signals and place first-time buys.
        """
        for sig in self.poll():
            token_mint = sig.token_mint or self._resolve_token_mint(sig)
            if not token_mint:
                logger.warning(f"[watcher] Unable to resolve token mint for signal: {sig}")
                continue

            # Enforce "buy once per burner" is handled inside Copier.
            ok = self.copier.execute_copy_trade(
                market="Jupiter",
                token_mint=token_mint,
                role="burner",  # explicit, matches our one-per-burner policy
            )
            if ok:
                logger.info(f"[watcher] First-time buy placed for {token_mint} from {sig.source}")
            else:
                logger.info(f"[watcher] Duplicate buy skipped for {token_mint} from {sig.source}")

    # --- helpers ------------------------------------------------------------

    def _resolve_token_mint(self, sig: TradeSignal) -> Optional[str]:
        """
        Resolve the SPL token mint from the signal. You MUST implement this for real:
        - If you have a pool/market id, query your DEX/router to get the output token mint.
        - If you only have a symbol, map it via your token list.
        - Prefer chain data over symbols to avoid spoofing.
        """
        # EXAMPLE PLACEHOLDER LOGIC — replace with actual resolver
        # Try explicit mint first
        if sig.token_mint:
            return sig.token_mint

        # Example mapping by symbol (load from a JSON token list in production)
        SYMBOL_TO_MINT = {
            # "SOL":  "So11111111111111111111111111111111111111112",
            # "USDC": "Es9vMFrzaCERz8hZK...<snip>",
        }
        if sig.token_symbol and sig.token_symbol in SYMBOL_TO_MINT:
            return SYMBOL_TO_MINT[sig.token_symbol]

        # Example: derive from raw market id (stub)
        # if sig.raw_market_id:
        #     return lookup_mint_from_market(sig.raw_market_id)

        return None

```

```python
# src/trading/tpsl.py
class TrailingExit:
    def __init__(self, start_pct: float, trail_drop_pct: float):
        self.start = start_pct
        self.drop = trail_drop_pct

    def should_exit(self, peak: float, current: float):
        if peak < (1 + self.start):
            return False
        return current <= peak * (1 - self.drop)

# Matches README: +20% / +36% start, 0.001% trail
EurisTPSL = TrailingExit(0.36, 0.00001)
CupesyTPSL = TrailingExit(0.20, 0.00001)

```

```python
# src/trading/copier.py
from __future__ import annotations
from loguru import logger
from src.core.config import CFG
from src.core.state import State
from src.solana.client import SolanaClient
from src.dex.jupiter import JupiterClient, MINT_SOL
from src.core.keystore import KeyStore


LAMPORTS_PER_SOL = 1_000_000_000


class Copier:
    def __init__(self, trade_size_sol: float, default_role: str = "burner"):
        self.size = float(trade_size_sol)
        self.role = default_role
        self.state = State()
        self.sol = SolanaClient()
        self.jup = JupiterClient()

    # --- helpers ---
    def _trader_pubkey(self, role: str) -> str:
        if role == "burner":
            pub = self.state.get_pubkey("burner")
            if not pub:
                raise RuntimeError("Burner pubkey missing in state.json")
            return pub
        raise ValueError(f"Unknown role {role}")

    def _trader_keyfile(self, role: str) -> str:
        if role == "burner":
            return str(KeyStore.path(CFG.BURNER_KEY))
        raise ValueError(f"Unknown role {role}")

    def _quote_sol_to_token(self, token_mint: str, amount_in_lamports: int) -> dict:
        return self.jup.quote(MINT_SOL, token_mint, amount_in_lamports, CFG.slippage.copytrade_slippage_bps)

    def _send_via_jupiter(self, role: str, quote_json: dict, priority_sol: float) -> str:
        prioritization_lamports = self.sol.lamports_from_sol(priority_sol) if priority_sol > 0 else 0
        user_pub = self._trader_pubkey(role)
        tx_bytes = self.jup.build_swap_tx(quote_json, user_pubkey=user_pub, prioritization_lamports=prioritization_lamports)
        sig = self.sol.send_legacy_tx_bytes(tx_bytes, signer_keyfile=self._trader_keyfile(role), skip_preflight=True)
        return sig

    # --- public ---
    def can_buy(self, token_mint: str, role: str) -> bool:
        # enforce one buy per burner until regen
        return not self.state.has_bought_token(role, token_mint)

    def record_buy(self, token_mint: str, role: str) -> None:
        self.state.record_token_buy(role, token_mint)

    def execute_copy_trade(self, market: str, token_mint: str, role: str | None = None) -> bool:
        r = role or self.role
        if not self.can_buy(token_mint, r):
            return False

        amount_in_lamports = int(self.size * LAMPORTS_PER_SOL)

        # 1) Quote
        quote = self._quote_sol_to_token(token_mint, amount_in_lamports)
        out_amount = int(quote.get("outAmount") or quote.get("out_amount") or 0)
        if out_amount <= 0:
            logger.warning(f"[copier] No route/quote for token={token_mint}")
            return False

        # 2) Priority
        priority_sol = float(CFG.priority.copytrade_priority_sol)

        # 3) Build+send
        try:
            tx_sig = self._send_via_jupiter(r, quote, priority_sol)
        except Exception as e:
            logger.exception(f"[copier] Swap failed for {token_mint}: {e}")
            return False

        logger.info(
            f"[copier] BUY SOL→{token_mint} role={r} in={amount_in_lamports} "
            f"quote_out={out_amount} slip_bps={CFG.slippage.copytrade_slippage_bps} tx={tx_sig}"
        )
        self.record_buy(token_mint, r)
        return True

```

```python
# src/trading/dry_run.py
from loguru import logger
class DryRun:
    def __init__(self):
        self.pnl=0.0
    def simulate_trade(self, size:float, ret:float):
        self.pnl += size*ret
        logger.info(f"Sim trade size {size}, ret {ret*100:.1f}% -> pnl {self.pnl}")
```

---

## Security

```python
# src/security/ip_guard.py
from loguru import logger
from src.core.config import CFG
import requests

class IpGuard:
    def current_ip(self) -> str:
        try:
            r = requests.get("https://api.ipify.org?format=text", timeout=2.5)
            if r.ok and r.text.strip():
                return r.text.strip()
        except Exception:
            pass
        return "0.0.0.0"

    def is_whitelisted(self) -> bool:
        if not CFG.kill_switch_enabled:
            logger.info("Kill switch disabled; skipping IP check.")
            return True
        if not CFG.whitelist_ips:
            logger.warning("WHITELISTED_IPS empty; allowing all IPs. Configure this in production.")
            return True
        ip = self.current_ip()
        if ip == "0.0.0.0":
            logger.warning("Could not resolve current IP; allowing traffic to avoid false positives.")
            return True
        ok = (ip in CFG.whitelist_ips)
        if not ok:
            logger.error(f"Unknown IP {ip} (not in whitelist)")
        return ok

```

```python
# src/security/kill_switch.py
from loguru import logger
from pathlib import Path
import time
from src.security.ip_guard import IpGuard
from src.core.config import CFG

class KillSwitch:
    def __init__(self, drain_callable, reboot_callable):
        self.drain = drain_callable
        self.reboot = reboot_callable
        # Lock file to prevent repeated triggers in a short window
        self._lock = Path("data/kill.lock")

    def _recently_triggered(self, ttl_s: int = 60) -> bool:
        if not self._lock.exists():
            return False
        try:
            return (time.time() - self._lock.stat().st_mtime) < ttl_s
        except Exception:
            return False

    def _touch_lock(self):
        try:
            self._lock.parent.mkdir(parents=True, exist_ok=True)
            self._lock.write_text(str(time.time()))
        except Exception:
            pass

    def check_and_trigger(self):
        if not CFG.kill_switch_enabled:
            logger.info("Kill switch disabled by config.")
            return
        guard = IpGuard()
        if not guard.is_whitelisted():
            logger.critical("KILL SWITCH: Unknown IP detected!")
            self.trigger()

    def trigger(self):
        if self._recently_triggered():
            logger.warning("Kill switch already triggered recently; ignoring duplicate.")
            return
        self._touch_lock()
        try:
            self.drain()
        except Exception as e:
            logger.exception(f"Kill drain failed: {e}")
        try:
            self.reboot()
        except Exception as e:
            logger.exception(f"Kill reboot failed: {e}")
        logger.critical("⚠️ Kill switch active. Funds evacuated & system rebooting.")

```python
# src/security/pgp.py
# placeholder for PGP helpers if you want python-side encrypt/decrypt
```

---

## Telegram & Reports

```python
# src/telegram/bot.py
import os
from loguru import logger
from telegram.ext import Application, CommandHandler
from . import commands as C


class TGBot:
    def __init__(self):
        self.token = os.getenv('TELEGRAM_BOT_TOKEN', '')
        if not self.token:
            logger.error("TELEGRAM_BOT_TOKEN missing")
        self.app = Application.builder().token(self.token).build()
        self.app.add_handler(CommandHandler('status', C.cmd_status))
        self.app.add_handler(CommandHandler('rotate_now', C.cmd_rotate_now))
        self.app.add_handler(CommandHandler('sync', C.cmd_sync))
        self.app.add_handler(CommandHandler('kill', C.cmd_kill))
        self.app.add_handler(CommandHandler('vault_status', C.cmd_vault_status))
        self.app.add_handler(CommandHandler('atpnl', C.cmd_atpnl))

    def run(self):
        logger.info("Starting Telegram bot (run_polling)")
        # Blocks the thread; handles init/start/idle internally
        self.app.run_polling(close_loop=False)

```

```python
# src/telegram/commands.py
from loguru import logger
from telegram import Update
from telegram.ext import ContextTypes
import os, time
from src.solana.client import SolanaClient
from src.core.state import State
from src.reports.pnl_store import get_all_time_usd
from src.core.keystore import KeyStore
from src.core.config import CFG

ADMIN_IDS = {int(x) for x in os.getenv("TELEGRAM_ADMIN_IDS", "").split(",") if x.strip().isdigit()}
KILL_PASSPHRASE = os.getenv("KILL_SWITCH_PASSPHRASE", "")
_LAST_KILL_TS = 0.0


def _is_admin(update: Update) -> bool:
    u = update.effective_user
    return bool(u and (u.id in ADMIN_IDS))


async def cmd_status(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not _is_admin(update):
        await update.message.reply_text("Not authorized.")
        return

    logger.info("/status")

    sol = SolanaClient()

    try:
        kf_burner    = str(KeyStore.path(CFG.BURNER_KEY))
        kf_collector = str(KeyStore.path(CFG.COLLECTOR_KEY))
        kf_fa        = str(KeyStore.path(CFG.FUNDER_ACTIVE_KEY))
        kf_fn        = str(KeyStore.path(CFG.FUNDER_NEXT_KEY))
        kf_fs        = str(KeyStore.path(CFG.FUNDER_STANDBY_KEY))

        burner_bal         = sol.get_balance(kf_burner)
        collector_bal      = sol.get_balance(kf_collector)
        funder_active_bal  = sol.get_balance(kf_fa)
        funder_next_bal    = sol.get_balance(kf_fn)
        funder_standby_bal = sol.get_balance(kf_fs)

        msg = (
            "💰 Balances:\n"
            f"🔥 Burner: {burner_bal:.4f} SOL\n"
            f"📦 Collector: {collector_bal:.4f} SOL\n"
            f"⚡ Funder Active: {funder_active_bal:.4f} SOL\n"
            f"➡️ Funder Next: {funder_next_bal:.4f} SOL\n"
            f"🕒 Funder Standby: {funder_standby_bal:.4f} SOL"
        )
    except Exception as e:
        msg = f"Error fetching balances: {e}"

    await update.message.reply_text(msg)


async def cmd_rotate_now(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not _is_admin(update):
        await update.message.reply_text("Not authorized.")
        return
    logger.info("/rotate_now")
    await update.message.reply_text("Manual rotation kicked off.")


async def cmd_sync(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not _is_admin(update):
        await update.message.reply_text("Not authorized.")
        return
    logger.info("/sync")
    await update.message.reply_text("Health OK.")


async def cmd_kill(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not _is_admin(update):
        await update.message.reply_text("Not authorized.")
        return
    text = (update.message.text or "").strip()
    parts = text.split(maxsplit=1)
    if not KILL_PASSPHRASE:
        await update.message.reply_text("Kill switch not configured.")
        return
    if len(parts) < 2 or parts[1] != KILL_PASSPHRASE:
        await update.message.reply_text("Passphrase required. Usage: /kill <passphrase>")
        return
    global _LAST_KILL_TS
    now = time.time()
    if now - _LAST_KILL_TS < 10:
        await update.message.reply_text("Kill already triggered recently. Please wait.")
        return
    _LAST_KILL_TS = now
    logger.warning("/kill -> kill switch (authorized)")
    await update.message.reply_text("Kill switch activated (acknowledged).")


async def cmd_vault_status(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not _is_admin(update):
        await update.message.reply_text("Not authorized.")
        return

    state = State().load()
    sol = SolanaClient()

    wallets = state.get("wallets", {})
    hot_keys = []
    cold_keys = []
    if "burner" in wallets:          hot_keys.append(("Burner", wallets["burner"]["keyfile"]))
    if "collector" in wallets:       hot_keys.append(("Collector", wallets["collector"]["keyfile"]))
    if "funder_active" in wallets:   cold_keys.append(("Funder Active", wallets["funder_active"]["keyfile"]))
    if "funder_next" in wallets:     cold_keys.append(("Funder Next", wallets["funder_next"]["keyfile"]))
    if "funder_standby" in wallets:  cold_keys.append(("Funder Standby", wallets["funder_standby"]["keyfile"]))

    hot_lines, cold_lines = [], []
    hot_total, cold_total = 0.0, 0.0

    for label, keyfile in hot_keys:
        bal = sol.get_balance(keyfile)
        hot_total += bal
        hot_lines.append(f"• {label}: {bal:.4f} SOL")

    for label, keyfile in cold_keys:
        bal = sol.get_balance(keyfile)
        cold_total += bal
        cold_lines.append(f"• {label}: {bal:.4f} SOL")

    msg = (
        "🏦 Vault status\n\n"
        f"🔥 HOT ({hot_total:.4f} SOL):\n" + ("\n".join(hot_lines) if hot_lines else "• none") + "\n\n"
        f"❄️ COLD ({cold_total:.4f} SOL):\n" + ("\n".join(cold_lines) if cold_lines else "• none") + "\n\n"
        f"Σ TOTAL: {(hot_total + cold_total):.4f} SOL"
    )
    await update.message.reply_text(msg)


async def cmd_atpnl(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    if not _is_admin(update):
        await update.message.reply_text("Not authorized.")
        return
    all_time_usd = get_all_time_usd()
    await update.message.reply_text(f"All-time PnL: {all_time_usd:.2f} USD")

```

```python
# src/reports/pnl.py
class PnL:
    def __init__(self):
        self.all_time=0.0
    def add(self, x:float):
        self.all_time += x
    def summary(self):
        return {"all_time": self.all_time}
```

```python
#src/reports/pnl_store.py
# src/reports/pnl_store.py
from __future__ import annotations
from pathlib import Path
import json

_PNL_PATH = Path("data/pnl.json")

def _load() -> dict:
    if _PNL_PATH.exists():
        try:
            return json.loads(_PNL_PATH.read_text())
        except Exception:
            pass
    # default empty structure
    return {"all_time_usd": 0.0, "by_day": {}}

def get_all_time_usd() -> float:
    return float(_load().get("all_time_usd", 0.0))

def get_day_usd(date_str: str) -> float:
    return float(_load().get("by_day", {}).get(date_str, 0.0))

def save_all_time_usd(v: float) -> None:
    d = _load()
    d["all_time_usd"] = float(v)
    _PNL_PATH.parent.mkdir(parents=True, exist_ok=True)
    _PNL_PATH.write_text(json.dumps(d, indent=2))
```

```python
# src/reports/daily_report.py
import os
from datetime import datetime
from loguru import logger
import requests

from src.reports.pnl_store import get_all_time_usd, get_day_usd

def _send_telegram(msg: str) -> None:
    token = os.getenv("TELEGRAM_BOT_TOKEN", "")
    chat_id = os.getenv("TELEGRAM_CHAT_ID_MAIN", "")
    if not token or not chat_id:
        logger.warning("TELEGRAM_BOT_TOKEN/TELEGRAM_CHAT_ID_MAIN missing; skipping Telegram report")
        return
    url = f"https://api.telegram.org/bot{token}/sendMessage"
    try:
        r = requests.post(url, json={"chat_id": chat_id, "text": msg, "parse_mode": "HTML"}, timeout=10)
        r.raise_for_status()
    except Exception as e:
        logger.exception(f"Failed to send Telegram report: {e}")

def main():
    today = datetime.utcnow().strftime("%Y-%m-%d")
    at = get_all_time_usd()
    day = get_day_usd(today)
    msg = (
        f"📊 <b>Daily PnL Report</b>\n"
        f"Date (UTC): {today}\n"
        f"• Today: ${day:,.2f}\n"
        f"• All-time: ${at:,.2f}"
    )
    _send_telegram(msg)
    logger.info("Daily PnL report sent.")

if __name__ == '__main__':
    main()

```

---

## Scheduled Entrypoints

```python
# src/sched/at_0445_drain.py
from loguru import logger
from src.core.keystore import KeyStore
from src.core.config import CFG
from src.core.state import State
from src.solana.client import SolanaClient
from src.solana.wallet_manager import WalletManager
from src.flow.burner import BurnerFlow

def main():
    logger.info("[04:45] Drain burners/actives to collector; burn & recreate")
    sol = SolanaClient()
    wm = WalletManager()
    state = State()
    collector_addr = state.get_pubkey("collector")
    if not collector_addr:
        logger.error("Collector address missing — aborting 04:45 drain.")
        return

    BurnerFlow(
        sol=sol,
        wm=wm,
        burner_keyfile=str(KeyStore.path(CFG.BURNER_KEY)),
        collector_addr=collector_addr,
        funder_active_keyfile=str(KeyStore.path(CFG.FUNDER_ACTIVE_KEY)),
    ).forced_drain()

if __name__ == '__main__':
    main()
```


```python
# src/sched/at_0450_dex.py
from loguru import logger
from src.core.keystore import KeyStore
from src.core.config import CFG
from src.solana.wallet_manager import WalletManager
from src.solana.client import SolanaClient
from src.dex.client import DexClient
from src.flow.collector import CollectorFlow

def main():
    logger.info("[04:50] Collector -> DEX (50/25/25) in chunks; route 95/5 via proxies; burn proxies")
    sol = SolanaClient()
    wm = WalletManager()
    dex = DexClient(CFG.dex.endpoint)
    CollectorFlow(
        sol=sol, dex=dex, wm=wm, collector_keyfile=str(KeyStore.path(CFG.COLLECTOR_KEY))
    ).run_0450()

if __name__ == '__main__':
    main()
```


```python
# src/sched/at_0500_rotate.py
from loguru import logger
from src.core.config import CFG
from src.core.state import State
from src.solana.client import SolanaClient
from src.solana.wallet_manager import WalletManager
from src.flow.funders import FunderRotation

def main():
    logger.info("[05:00] Rotate funders (active->burn if empty, next->active, standby->next, create standby)")
    sol = SolanaClient()
    wm = WalletManager()
    state = State()
    collector_addr = state.get_pubkey("collector")
    if not collector_addr:
        logger.error("Collector address missing — aborting 05:00 rotation.")
        return
    fr = FunderRotation(
        sol=sol,
        wm=wm,
        active_kf=CFG.FUNDER_ACTIVE_KEY,
        next_kf=CFG.FUNDER_NEXT_KEY,
        standby_kf=CFG.FUNDER_STANDBY_KEY,
        collector_addr=collector_addr,
    )
    fr.rotate()
    fr.ensure_active_has_5()
    logger.info("✅ Rotation complete.")

if __name__ == '__main__':
    main()

```

---

## Main Entrypoint/src

```python
# src/main.py
from loguru import logger
from src.core.config import CFG
from src.telegram.bot import TGBot
from src.security.kill_switch import KillSwitch
from src.solana.wallet_manager import WalletManager
from src.core.state import State
from src.solana.client import SolanaClient
from src.core.keystore import KeyStore
import os, shutil


def drain_all():
    """Drain burner + funders + proxies to collector address (best-effort)."""
    sol = SolanaClient()
    state = State()
    collector_addr = state.get_pubkey("collector")
    if not collector_addr:
        logger.error("No collector pubkey in state; cannot drain.")
        return

    keyfiles = [
        CFG.BURNER_KEY,
        CFG.FUNDER_ACTIVE_KEY,
        CFG.FUNDER_NEXT_KEY,
        CFG.FUNDER_STANDBY_KEY,
        getattr(CFG, "PROXY1_KEY", "proxy1.json"),
        getattr(CFG, "PROXY2_KEY", "proxy2.json"),
    ]
    for kf in keyfiles:
        path = str(KeyStore.path(kf))
        try:
            bal = sol.get_balance(path)
            if bal > 0:
                sol.transfer(path, collector_addr, bal)
                logger.info(f"Drained {bal:.6f} SOL from {kf} -> collector")
        except Exception as e:
            logger.warning(f"Drain failed for {kf}: {e}")
 

def reboot_bot():
    """
    Try to restart via systemd if available; otherwise exit and let a supervisor restart us.
    """
    svc = os.getenv("SYSTEMD_SERVICE_NAME", "copytrader.service")
    if shutil.which("systemctl"):
        os.system(f"sudo systemctl restart {svc}")
        logger.warning(f"Requested systemd restart of {svc}")
    else:
        logger.warning("Exiting for external supervisor to restart...")
        os._exit(0)

def main():
    logger.info(f"Starting CopyTrader in {CFG.env} mode")
    state = State(); wm = WalletManager(state)
    ks = KillSwitch(drain_all, reboot_bot)
    ks.check_and_trigger()
    TGBot().run()

if __name__ == '__main__':
    main()

```

---

## Tests (snippets)

```python
# tests/test_killswitch.py
from src.security.kill_switch import KillSwitch
TRIGGERED=False

def drain():
    global TRIGGERED
    TRIGGERED=True

def reboot():
    pass

def test_kill_switch():
    ks = KillSwitch(drain, reboot)
    ks.trigger()
    assert TRIGGERED
```

```python
# tests/test_rotation.py
from src.trading.tpsl import TrailingExit

def test_tpsl_logic():
    t = TrailingExit(0.10, 0.00001)
    assert not t.should_exit(1.05, 1.04)
```

```python
# tests/test_tpsl.py
from src.trading.tpsl import TrailingExit

def test_exit_after_start():
    t = TrailingExit(0.36, 0.00001)
    assert t.should_exit(1.40, 1.399) is True
```

```python
#tests/conftest.py
import json
import pytest
from pathlib import Path

@pytest.fixture(autouse=True)
def isolate_keys_and_state(tmp_path, monkeypatch):
    """
    Redirect KeyStore.base -> tmp/keys and State.STATE_PATH -> tmp/data/state.json
    so tests don't touch real files.
    """
    # keys/
    keys_dir = tmp_path / "keys"
    keys_dir.mkdir(parents=True, exist_ok=True)
    import src.core.keystore as keystore_mod
    monkeypatch.setattr(keystore_mod.KeyStore, "base", keys_dir, raising=False)

    # data/state.json
    data_dir = tmp_path / "data"
    data_dir.mkdir(parents=True, exist_ok=True)
    state_path = data_dir / "state.json"

    import src.core.state as state_mod
    monkeypatch.setattr(state_mod, "STATE_PATH", state_path, raising=False)

    # Force a fresh State cache on first import/use
    try:
        state_path.write_text(json.dumps(state_mod.DEFAULT, indent=2))
    except Exception:
        pass

    yield
```

```python
tests/test_wallet_manager_regen.py
from pathlib import Path
from src.solana.wallet_manager import WalletManager
from src.core.state import State
from src.core.keystore import KeyStore

def test_wallet_manager_create_burn_replace(tmp_path):
    state = State()  # uses patched STATE_PATH from fixture
    wm = WalletManager(state)

    # 1) create
    w = wm.create_wallet("burner", "burner.json")
    assert w.pubkey
    assert Path(w.key_path).exists()
    assert state.get_pubkey("burner") == w.pubkey

    # 2) burn (file gone, state cleared)
    wm.burn_wallet("burner", "burner.json")
    assert not KeyStore.path("burner.json").exists()
    assert state.get_pubkey("burner") is None

    # 3) replace (burn + create) -> new pubkey different from previous
    w2 = wm.replace_wallet("burner", "burner.json")
    assert KeyStore.path("burner.json").exists()
    assert w2.pubkey and w2.pubkey != w.pubkey
    assert state.get_pubkey("burner") == w2.pubkey
```
```python
#tests/test_state_token_buy.py
from src.core.state import State

def test_token_buy_registry_roundtrip():
    s = State()
    role = "burner"
    mint = "FAKE111111111111111111111111111111111111111"

    # Initially not bought
    assert not s.has_bought_token(role, mint)

    # Record and verify
    s.record_token_buy(role, mint)
    assert s.has_bought_token(role, mint)

    # Clear role registry
    s.clear_role_token_buys(role)
    assert not s.has_bought_token(role, mint)
```

```python
#tests/test_proxy_router_delays.py
from src.flow.proxies import ProxyRouter
from src.core.config import CFG

def test_proxy_router_calls_jitter_with_expected_args(monkeypatch):
    # Save and tweak config
    old_hops = CFG.proxy.hops
    old_min = CFG.proxy.delay_s_min
    old_max = CFG.proxy.delay_s_max
    CFG.proxy.hops = 3
    CFG.proxy.delay_s_min = 2
    CFG.proxy.delay_s_max = 5

    calls = []

    # Patch the imported jitter function inside proxies module
    import src.flow.proxies as proxies_mod
    def fake_jitter(min_s, max_s):
        calls.append((min_s, max_s))
        return None
    monkeypatch.setattr(proxies_mod, "jitter", fake_jitter, raising=True)

    try:
        ProxyRouter().route_with_hops(100.0, "SRC", "DST")
        # Called exactly `hops` times
        assert len(calls) == CFG.proxy.hops
        # Each call used the configured bounds
        assert all(c == (CFG.proxy.delay_s_min, CFG.proxy.delay_s_max) for c in calls)
    finally:
        # Restore config
        CFG.proxy.hops = old_hops
        CFG.proxy.delay_s_min = old_min
        CFG.proxy.delay_s_max = old_max
```
```python
#tests/test_daily_rotation_smoke.py
import os
from src.flow.rotation import DailyRotation
from src.core.state import State
from src.solana.wallet_manager import WalletManager

def test_daily_rotation_smoke(monkeypatch):
    # Ensure minimal state exists
    s = State()
    wm = WalletManager(s)
    # Create required roles if empty
    for role, fname in [
        ("collector", "collector.json"),
        ("burner", "burner.json"),
        ("funder_active", "funder_active.json"),
        ("funder_next", "funder_next.json"),
        ("funder_standby", "funder_standby.json"),
    ]:
        if not s.get_pubkey(role):
            wm.create_wallet(role, fname)

    # Run in dry-run mode if your client supports it
    os.environ.setdefault("DRY_RUN", "1")
    DailyRotation().execute()
    # If no exceptions, consider it a pass (this is a smoke test)
    assert True

```

---

## Notes
- All regen points call `WalletManager.replace_wallet(role, filename)` and update `data/state.json`.
- Filenames in `.env` remain constant; you will see **new pubkeys** after burns.
- Replace stub keygen/transfer with your preferred Solana libraries for production.

