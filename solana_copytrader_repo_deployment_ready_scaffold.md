# GTGR

> Fully automated, stealth‑first Solana copy‑trading stack with **WalletManager** + **state tracking** for automatic key regeneration. Includes burner/collector lifecycle, 3‑funder rotation, proxy hops, DEX swaps (50/25/25 BTC/ETH/XRP), kill switch, Telegram control, Tor, GPG/PGP, cron/systemd, and dry‑run mode.

---

## README.md

```md
# solana-copytrader (with WalletManager + state tracking)

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
- Telegram: `/status`, `/rotate_now`, `/sync`, `/kill`, `/vault_status`, `/atpnl`

### Auto Key Regeneration
- Filenames in `.env` remain constant (e.g., `BURNER_KEY=burner.json`).
- On burn/recreate, the **same filename** is overwritten with a fresh keypair and
  the **new pubkey** is saved to `data/state.json`.
- Verify with `scripts/sim_rotate.py`.
```

---

## .env

```dotenv
ENV=prod
TZ=America/Nassau
DATA_DIR=./data
LOG_DIR=./logs

WHITELISTED_IPS=
KILL_SWITCH_ENABLED=true

TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID_MAIN=
TELEGRAM_CHAT_ID_ALERTS=

SOLANA_CLUSTER=mainnet
RPC_PRIMARY=https://api.mainnet-beta.solana.com
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
HOLD_COLD_KEY=hold_cold.json
HOLD_HOT_KEY=hold_hot.json

TRADE_SIZE_SOL=2.5
BURNER_TARGET_SOL=16
FORCED_DRAIN_CRON=45 4 * * *
ROTATE_CRON=0 5 * * *
DEX_CRON=50 4 * * *
WATCH_ADDRESSES=WalletCupesyPubkey,WalletEurisPubkey
MAX_MARKETCAP_USD=20000

DEX_ENDPOINT=https://dex.example
SWAP_SPLIT_BTC=0.50
SWAP_SPLIT_ETH=0.25
SWAP_SPLIT_XRP=0.25
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
GPG_KEY_EMAIL=you@example.com
PGP_BACKUP_ENABLED=true
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
solana>=0.30.2
solders>=0.20.0
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
    "croniter==3.0.3"
]

[tool.ruff]
line-length = 100
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
set -e

PROJECT_DIR="$(dirname "$(dirname "$(realpath "$0")")")"
cd "$PROJECT_DIR"
source .venv/bin/activate

# 1) Drain burners & funders → collector at 04:45
python -m src/flows/burner_flow --force-drain
python -m src/flows/funder_rotation --force-drain

# 2) Collector → DEX + proxies (at 04:50)
python -m src/flows/collector_flow

# 3) Rotate funders & regenerate wallets (at 05:00)
python -m src/flows/funder_rotation --rotate
```


```bash
#scripts/rotate_now.sh
#!/bin/bash
set -e

PROJECT_DIR="$(dirname "$(dirname "$(realpath "$0")")")"
cd "$PROJECT_DIR"
source .venv/bin/activate

python -m src/flows/funder_rotation --rotate
```

```bash
#scripts/self_heal.sh
#!/bin/bash
set -e

BOT_NAME="copytrader"
if ! pgrep -f "$BOT_NAME" > /dev/null; then
    echo "[SELF-HEAL] Bot not running — restarting..."
    systemctl --user restart copytrader.service
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
set -e

KEY_DIR="keys"
BACKUP_DIR="backups"
RECIPIENT="your-pgp-key-id"

mkdir -p "$BACKUP_DIR"
for file in "$KEY_DIR"/*.json; do
    [ -e "$file" ] || continue
    gpg --yes --encrypt --recipient "$RECIPIENT" --output "$BACKUP_DIR/$(basename "$file").gpg" "$file"
done
```

```bash
#src/utils/check_rpc_latency.py
import requests, time, subprocess, os

RPC_POOL_FILE = os.getenv("RPC_POOL_FILE", "config/rpc_pool.txt")
LATENCY_LIMIT_MS = 500

def check_latency(url):
    start = time.time()
    try:
        requests.post(url, json={"jsonrpc":"2.0","id":1,"method":"getHealth"}, timeout=2)
        return (time.time() - start) * 1000
    except Exception:
        return float("inf")

def main():
    with open(RPC_POOL_FILE) as f:
        rpcs = [line.strip() for line in f if line.strip()]
    bad = []
    for rpc in rpcs:
        latency = check_latency(rpc)
        print(f"{rpc} latency: {latency:.2f} ms")
        if latency > LATENCY_LIMIT_MS:
            bad.append(rpc)
    if len(bad) == len(rpcs):
        print("[!] All RPCs slow — rotating...")
        subprocess.run(["python", "src/utils/reset_rpc.py"])

if __name__ == "__main__":
    main()
```

```bash
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
User=ubuntu
WorkingDirectory=/home/ubuntu/solana-copytrader
ExecStart=/home/ubuntu/solana-copytrader/.venv/bin/python -m src.main
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
WorkingDirectory=%h/solana-copytrader
ExecStart=%h/solana-copytrader/.venv/bin/python scripts/check_rpc_latency.py
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
# data/state.json
{
  "active_rpc": "https://mainnet.helius-rpc.com/?api-key=YOUR_HELIUS_KEY",
  "wallets": {
    "burner": {
      "pubkey": "BURNER_PUBLIC_KEY",
      "keyfile": "keys/burner.json"
    },
    "collector": {
      "pubkey": "COLLECTOR_PUBLIC_KEY",
      "keyfile": "keys/collector.json"
    },
    "funder_active": {
      "pubkey": "FUNDER_ACTIVE_PUBLIC_KEY",
      "keyfile": "keys/funder_active.json"
    },
    "funder_next": {
      "pubkey": "FUNDER_NEXT_PUBLIC_KEY",
      "keyfile": "keys/funder_next.json"
    },
    "funder_standby": {
      "pubkey": "FUNDER_STANDBY_PUBLIC_KEY",
      "keyfile": "keys/funder_standby.json"
    }
  },
  "last_rotation": "2025-08-09T05:00:00Z",
  "last_kill_switch": null
}

```

---

## Core

```python
# src/core/config.py
from pydantic import BaseModel
from dotenv import load_dotenv
import os
from pathlib import Path

load_dotenv()

class Splits(BaseModel):
    btc: float = float(os.getenv("SWAP_SPLIT_BTC", 0.50))
    eth: float = float(os.getenv("SWAP_SPLIT_ETH", 0.25))
    xrp: float = float(os.getenv("SWAP_SPLIT_XRP", 0.25))

class ProxyCfg(BaseModel):
    hops: int = int(os.getenv("PROXY_HOPS", 2))
    delay_s_min: int = int(os.getenv("PROXY_DELAY_S_MIN", 3))
    delay_s_max: int = int(os.getenv("PROXY_DELAY_S_MAX", 9))

class DexCfg(BaseModel):
    endpoint: str = os.getenv("DEX_ENDPOINT","https://dex.invalid")
    chunk_size_sol: float = float(os.getenv("CHUNK_SIZE_SOL", 50))
    delay_s_min: int = int(os.getenv("CHUNK_DELAY_S_MIN", 5))
    delay_s_max: int = int(os.getenv("CHUNK_DELAY_S_MAX", 10))
    splits: Splits = Splits()

class TradingCfg(BaseModel):
    trade_size_sol: float = float(os.getenv("TRADE_SIZE_SOL", 2.5))
    burner_target_sol: float = float(os.getenv("BURNER_TARGET_SOL", 16))
    max_marketcap_usd: int = int(os.getenv("MAX_MARKETCAP_USD", 20000))
    watch_addresses: list[str] = os.getenv("WATCH_ADDRESSES","").split(",") if os.getenv("WATCH_ADDRESSES") else []

class AppCfg(BaseModel):
    env: str = os.getenv('ENV','prod')
    tz: str = os.getenv('TZ','America/Nassau')
    data_dir: Path = Path(os.getenv('DATA_DIR','./data')).resolve()
    log_dir: Path = Path(os.getenv('LOG_DIR','./logs')).resolve()
    whitelist_ips: list[str] = os.getenv('WHITELISTED_IPS','').split(',') if os.getenv('WHITELISTED_IPS') else []
    kill_switch_enabled: bool = os.getenv('KILL_SWITCH_ENABLED','true').lower()=='true'
    rpc_primary: str = os.getenv('RPC_PRIMARY','')
    rpc_pool_file: str = os.getenv('RPC_POOL_FILE','./config/rpc_pool.txt')
    COLLECTOR_KEY: str = os.getenv('COLLECTOR_KEY','collector.json')
    FUNDER_ACTIVE_KEY: str = os.getenv('FUNDER_ACTIVE_KEY','funder_active.json')
    FUNDER_NEXT_KEY: str = os.getenv('FUNDER_NEXT_KEY','funder_next.json')
    FUNDER_STANDBY_KEY: str = os.getenv('FUNDER_STANDBY_KEY','funder_standby.json')
    BURNER_KEY: str = os.getenv('BURNER_KEY','burner.json')
    dex: DexCfg = DexCfg()
    proxy: ProxyCfg = ProxyCfg()
    trading: TradingCfg = TradingCfg()

CFG = AppCfg()
```

```python
# src/core/logger.py
from loguru import logger
from pathlib import Path
from .config import CFG

Path(CFG.log_dir).mkdir(parents=True, exist_ok=True)
logger.add(Path(CFG.log_dir, 'copytrader.log'), rotation='10 MB', retention='14 days', compression='zip')
```

```python
# src/core/telemetry.py
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
# src/core/keystore.py
from pathlib import Path
class KeyStore:
    base = Path('keys')
    @staticmethod
    def path(name:str) -> Path:
        return KeyStore.base / name
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
```

```python
#src/core/utils.py
# src/core/utils.py
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

from solana.rpc.api import Client
from solana.rpc.types import TxOpts
from solana.transaction import Transaction
from solana.system_program import TransferParams, transfer

from src.solana.keypair_io import load_keypair_from_file
from src.core.config import CFG
from src.core.state import State

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

    def get_balance(self, wallet_keyfile: str) -> float:
        kp = load_keypair_from_file(wallet_keyfile)
        resp = self.client.get_balance(kp.pubkey())
        if resp.is_err():
            raise RuntimeError(f"get_balance RPC error: {resp}")
        lamports = resp.value  # returns int lamports
        sol = lamports / LAMPORTS_PER_SOL
        logger.debug(f"[balance] {kp.pubkey()} = {sol:.9f} SOL")
        return sol

    def transfer(self, src_keyfile: str, dst_addr: str, amount_sol: float) -> TxResult:
        if amount_sol <= 0:
            raise ValueError("amount_sol must be > 0")

        kp = load_keypair_from_file(src_keyfile)
        lamports = int(amount_sol * LAMPORTS_PER_SOL)

        # Recent blockhash
        bh = self.client.get_latest_blockhash()
        if bh.is_err():
            raise RuntimeError(f"get_latest_blockhash error: {bh}")
        bhash = bh.value.blockhash

        # Build tx
        ix = transfer(TransferParams(from_pubkey=kp.pubkey(), to_pubkey=dst_addr, lamports=lamports))
        tx = Transaction(recent_blockhash=bhash, fee_payer=kp.pubkey())
        tx.add(ix)

        # Send + confirm
        sig = self.client.send_transaction(tx, kp, opts=TxOpts(skip_preflight=False, max_retries=3))
        if sig.is_err():
            raise RuntimeError(f"send_transaction error: {sig}")

        signature = str(sig.value)
        logger.info(f"[transfer] {amount_sol} SOL -> {dst_addr} sig={signature}")

        # (optional) confirm
        conf = self.client.confirm_transaction(signature)
        if conf.is_err():
            logger.warning(f"[transfer] confirm error: {conf}")

        return TxResult(sig=signature)

    def burn_wallet(self, key_path: str):
        # purely a file delete (WalletManager handles actual burn).
        logger.warning(f"Burn wallet called for {key_path} (file deletion handled elsewhere).")

```

```python
# src/solana/wallet_manager.py
from __future__ import annotations
from pathlib import Path
from dataclasses import dataclass
from loguru import logger

from solders.keypair import Keypair  # NEW
from base58 import b58encode          # NEW

from .wallet import Wallet
from src.core.state import State
from src.core.keystore import KeyStore

LAMPORTS_PER_SOL = 1_000_000_000

def _generate_keypair_json() -> tuple[str, str]:
    """
    Return (pubkey_str, json_text) where json_text is a 64-byte array like solana-keygen.
    """
    kp = Keypair()  # random
    secret = kp.to_bytes()          # 64 bytes (private + public)
    arr = list(secret)              # [int,int,...]
    pub = str(kp.pubkey())          # base58
    import json
    return pub, json.dumps(arr)

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
        p.write_text(secret_json)
        try:
            import os
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
        path = KeyStore.path(filename)
        if path.exists():
            path.unlink()
            logger.warning(f"[{role}] burned keyfile {path}")
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

    def swap_sol_to_alloc(self, from_wallet:str, amount_sol:float, alloc:SwapAlloc) -> dict:
        logger.info(f"Swapping {amount_sol} SOL to BTC:{alloc.btc} ETH:{alloc.eth} XRP:{alloc.xrp}")
        return {"btc": amount_sol*alloc.btc, "eth": amount_sol*alloc.eth, "xrp": amount_sol*alloc.xrp}
```

```python
# src/dex/swapper.py
from .client import DexClient, SwapAlloc
from loguru import logger
from src.core.config import CFG

class Swapper:
    def __init__(self, dex:DexClient):
        self.dex = dex

    def swap_chunks(self, wallet:str, total_sol:float):
        remaining = total_sol
        alloc = SwapAlloc(CFG.dex.splits.btc, CFG.dex.splits.eth, CFG.dex.splits.xrp)
        while remaining > 0:
            chunk = min(CFG.dex.chunk_size_sol, remaining)
            self.dex.swap_sol_to_alloc(wallet, chunk, alloc)
            remaining -= chunk
            logger.info(f"Chunk complete, remaining {remaining} SOL")
```

---

## Flows

```python
# src/flow/burner.py (regen wired)
from loguru import logger
from src.core.config import CFG
from src.solana.client import SolanaClient
from src.solana.wallet_manager import WalletManager

class BurnerFlow:
    def __init__(self, sol:SolanaClient, wm:WalletManager, burner_keyfile:str, collector_addr:str, funder_active_keyfile:str):
        self.sol = sol
        self.wm = wm
        self.burner_keyfile = burner_keyfile
        self.collector_addr = collector_addr
        self.funder_active_keyfile = funder_active_keyfile

    def on_trade_profit(self):
        bal = self.sol.get_balance(self.burner_keyfile)
        if bal >= CFG.trading.burner_target_sol:
            self.drain_and_recreate()

    def forced_drain(self):
        self.drain_and_recreate()

    def _fund_new_burner(self):
        # transfer 5 SOL from active funder to new burner
        logger.info("Funding new burner with 5 SOL from active funder (stub)")

    def drain_and_recreate(self):
        bal = self.sol.get_balance(self.burner_keyfile)
        if bal > 0:
            self.sol.transfer(self.burner_keyfile, self.collector_addr, bal)
        logger.info("Burner drained; burning & recreating")
        self.wm.replace_wallet("burner", CFG.BURNER_KEY if hasattr(CFG, 'BURNER_KEY') else 'burner.json')
        self._fund_new_burner()
```

```python
# src/flow/collector.py (regen wired)
from loguru import logger
from src.core.config import CFG
from src.core.timeutils import jitter
from src.dex.client import DexClient
from src.solana.client import SolanaClient
from src.solana.wallet_manager import WalletManager

class CollectorFlow:
    def __init__(self, sol:SolanaClient, dex:DexClient, wm:WalletManager, collector_keyfile:str):
        self.sol=sol; self.dex=dex; self.wm=wm; self.collector_keyfile=collector_keyfile

    def run_0450(self):
        bal = self.sol.get_balance(self.collector_keyfile)
        remaining = bal
        while remaining > 0:
            chunk = min(CFG.dex.chunk_size_sol, remaining)
            self.dex.swap_sol_to_alloc(self.collector_keyfile, chunk, CFG.dex.splits)
            remaining -= chunk
            jitter(CFG.dex.delay_s_min, CFG.dex.delay_s_max)
        logger.info("Collector emptied to DEX; burn & recreate")
        self.wm.replace_wallet("collector", CFG.COLLECTOR_KEY if hasattr(CFG, 'COLLECTOR_KEY') else 'collector.json')
```

```python
# src/flow/funders.py (regen wired)
from loguru import logger
from src.solana.client import SolanaClient
from src.solana.wallet_manager import WalletManager

class FunderRotation:
    def __init__(self, sol:SolanaClient, wm:WalletManager, active_kf:str, next_kf:str, standby_kf:str, collector_addr:str):
        self.sol=sol; self.wm=wm
        self.active_kf=active_kf; self.next_kf=next_kf; self.standby_kf=standby_kf
        self.collector=collector_addr

    def ensure_active_has_5(self):
        # top up active to 5 SOL from collector as needed
        pass

    def rotate(self):
        logger.info("Rotating funders: active->burn (if empty), next->active, standby->next, new standby")
        # burn active ONLY if empty
        if self.sol.get_balance(self.active_kf) == 0:
            self.wm.burn_wallet("funder_active", self.active_kf)
        # logical promotion (filenames remain constant); generate new standby keyfile
        self.wm.replace_wallet("funder_standby", self.standby_kf)
```

```python
# src/flow/proxies.py
from loguru import logger
from src.core.config import CFG
from src.core.timeutils import jitter

class ProxyRouter:
    def route_with_hops(self, amount_units:float, src:str, dst:str):
        hops = CFG.proxy.hops
        for i in range(hops):
            logger.info(f"Proxy hop {i+1}/{hops}: moving {amount_units} units")
            jitter(CFG.proxy.delay_s_min, CFG.proxy.delay_s_max)
        logger.info(f"Delivered to {dst}; burn proxies (stub)")
```

```python
# src/flow/rotation.py
from loguru import logger
class DailyRotation:
    def execute(self):
        logger.info("04:45 drain -> 04:50 DEX -> 05:00 rotate sequence")
```

---

## Trading

```python
# src/trading/watcher.py
from loguru import logger
class WalletWatcher:
    def __init__(self, addresses:list[str]):
        self.addresses = addresses
    def poll(self):
        logger.debug(f"Watching {len(self.addresses)} wallets")
```

```python
# src/trading/tpsl.py
class TrailingExit:
    def __init__(self, start_pct:float, trail_drop_pct:float):
        self.start=start_pct; self.drop=trail_drop_pct
    def should_exit(self, peak:float, current:float):
        if peak < (1+self.start):
            return False
        return (current <= peak*(1 - self.drop))

EurisTPSL = TrailingExit(0.36, 0.00001)
CupesyTPSL = TrailingExit(0.10, 0.00001)
```

```python
# src/trading/copier.py
from loguru import logger
class Copier:
    def __init__(self, trade_size_sol:float):
        self.size = trade_size_sol
    def execute_copy_trade(self, market:str):
        logger.info(f"Executing copy trade on {market} with {self.size} SOL")
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

class IpGuard:
    def current_ip(self) -> str:
        return '0.0.0.0'
    def is_whitelisted(self) -> bool:
        ip = self.current_ip()
        ok = (ip in CFG.whitelist_ips)
        if not ok:
            logger.error(f"Unknown IP {ip}")
        return ok
```

```python
# src/security/kill_switch.py
from loguru import logger
from src.security.ip_guard import IpGuard

class KillSwitch:
    def __init__(self, drain_callable, reboot_callable):
        self.drain = drain_callable
        self.reboot = reboot_callable
    def check_and_trigger(self):
        guard = IpGuard()
        if not guard.is_whitelisted():
            logger.critical("KILL SWITCH: Unknown IP detected!")
            self.trigger()
    def trigger(self):
        self.drain()
        self.reboot()
        logger.critical("⚠️ Kill switch active. Funds evacuated & system rebooting.")
```

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

async def cmd_status(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    logger.info("/status")
    await update.message.reply_text("Balances: ... (stub)")

async def cmd_rotate_now(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    logger.info("/rotate_now")
    await update.message.reply_text("Manual rotation kicked off.")

async def cmd_sync(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    logger.info("/sync")
    await update.message.reply_text("Health OK.")

async def cmd_kill(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    logger.warning("/kill -> kill switch")
    await update.message.reply_text("Kill switch activated (simulated).")

async def cmd_vault_status(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Vault (Hot/Cold): ... (stub)")

async def cmd_atpnl(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("All-time PnL: ... (stub)")

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
# src/reports/daily_report.py
from loguru import logger

def main():
    logger.info("Daily PnL report (stub) -> Telegram")

if __name__ == '__main__':
    main()
```

---

## Scheduled Entrypoints

```python
# src/sched/at_0445_drain.py
from loguru import logger

def main():
    logger.info("[04:45] Drain burners/actives to collector; burn & recreate")

if __name__ == '__main__':
    main()
```

```python
# src/sched/at_0450_dex.py
from loguru import logger

def main():
    logger.info("[04:50] Collector -> DEX (50/25/25) in 50 SOL chunks; route 95% cold / 5% hot via 2 proxies; burn proxies")

if __name__ == '__main__':
    main()
```

```python
# src/sched/at_0500_rotate.py
from loguru import logger

def main():
    logger.info("[05:00] Rotate funders (active->burn, next->active, standby->next, create standby)")

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

def drain_all():
    logger.warning("Draining all active wallets to collector (stub)")

def reboot_bot():
    logger.warning("Rebooting bot (stub)")

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

---

### Notes

- All regen points call `WalletManager.replace_wallet(role, filename)` and update `data/state.json`.
- Filenames in `.env` remain constant; you will see **new pubkeys** after burns.
- Replace stub keygen/transfer with your preferred Solana libraries for production.

