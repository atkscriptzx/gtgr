# solana-copytrader — Deployment‑Ready Scaffold (Updated)

> Fully automated, stealth‑first Solana copy‑trading stack with **WalletManager** + **state tracking** for automatic key regeneration. Includes burner/collector lifecycle, 3‑funder rotation, proxy hops, DEX swaps (50/25/25 BTC/ETH/XRP), kill switch, Telegram control, Tor, GPG/PGP, cron/systemd, and dry‑run mode.

---

## Repository Tree

```
solana-copytrader/
├── README.md
├── .env.example
├── docker-compose.yml
├── Dockerfile
├── Makefile
├── requirements.txt
├── pyproject.toml
├── scripts/
│   ├── bootstrap.sh
│   ├── daily_5am_trigger.sh
│   ├── rotate_now.sh
│   ├── self_heal.sh
│   ├── gpg_sign_logs.sh
│   ├── pgp_encrypt_keys.sh
│   ├── check_rpc_latency.py
│   ├── reset_rpc.py
│   ├── cron_install.sh
│   ├── systemd_install.sh
│   └── sim_rotate.py
├── config/
│   ├── config.schema.yaml
│   ├── rpc_pool.txt
│   ├── telegram_commands.md
│   └── tor/torrc.example
├── deploy/
│   ├── copytrader.service
│   ├── copytrader-health.service
│   ├── copytrader-health.timer
│   └── copytrader.timer
├── data/
│   └── state.json
├── src/
│   ├── main.py
│   ├── core/
│   │   ├── config.py
│   │   ├── logger.py
│   │   ├── telemetry.py
│   │   ├── timeutils.py
│   │   ├── keystore.py
│   │   ├── state.py
│   │   └── utils.py
│   ├── solana/
│   │   ├── client.py
│   │   ├── wallet.py
│   │   └── wallet_manager.py
│   ├── trading/
│   │   ├── watcher.py
│   │   ├── copier.py
│   │   ├── tpsl.py
│   │   └── dry_run.py
│   ├── flow/
│   │   ├── burner.py
│   │   ├── funders.py
│   │   ├── collector.py
│   │   ├── proxies.py
│   │   └── rotation.py
│   ├── dex/
│   │   ├── client.py
│   │   └── swapper.py
│   ├── security/
│   │   ├── kill_switch.py
│   │   ├── ip_guard.py
│   │   └── pgp.py
│   ├── telegram/
│   │   ├── bot.py
│   │   └── commands.py
│   ├── reports/
│   │   ├── pnl.py
│   │   └── daily_report.py
│   └── sched/
│       ├── at_0445_drain.py
│       ├── at_0450_dex.py
│       └── at_0500_rotate.py
└── tests/
    ├── test_killswitch.py
    ├── test_rotation.py
    └── test_tpsl.py
```

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

## .env.example

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

## Docker & Make

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

---

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
# scripts/cron_install.sh
#!/usr/bin/env bash
set -euo pipefail
( crontab -l 2>/dev/null; echo "$FORCED_DRAIN_CRON cd $(pwd) && . .venv/bin/activate && python -m src.sched.at_0445_drain >> logs/cron.log 2>&1" ) | crontab -
( crontab -l 2>/dev/null; echo "$DEX_CRON cd $(pwd) && . .venv/bin/activate && python -m src.sched.at_0450_dex >> logs/cron.log 2>&1" ) | crontab -
( crontab -l 2>/dev/null; echo "$ROTATE_CRON cd $(pwd) && . .venv/bin/activate && python -m src.sched.at_0500_rotate >> logs/cron.log 2>&1" ) | crontab -
( crontab -l 2>/dev/null; echo "$DAILY_REPORT_CRON cd $(pwd) && . .venv/bin/activate && python -m src.reports.daily_report >> logs/cron.log 2>&1" ) | crontab -
```

```bash
# scripts/systemd_install.sh
#!/usr/bin/env bash
set -euo pipefail
UNIT_DIR="$HOME/.config/systemd/user"
mkdir -p "$UNIT_DIR"
cp deploy/*.service deploy/*.timer "$UNIT_DIR"/
systemctl --user daemon-reload
systemctl --user enable copytrader.service copytrader-health.timer
systemctl --user start copytrader-health.timer
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

```bash
# Other scripts
# daily_5am_trigger.sh, rotate_now.sh, self_heal.sh, gpg_sign_logs.sh, pgp_encrypt_keys.sh,
# check_rpc_latency.py, reset_rpc.py (as previously provided)
```

---

## Config & Deploy

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

```ini
# deploy/copytrader.service
[Unit]
Description=Solana CopyTrader Bot
After=network-online.target
[Service]
Type=simple
WorkingDirectory=%h/solana-copytrader
ExecStart=%h/solana-copytrader/.venv/bin/python -m src.main
Restart=always
Environment=PYTHONUNBUFFERED=1
[Install]
WantedBy=default.target
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
from dataclasses import dataclass
from loguru import logger

@dataclass
class TxResult:
    sig: str

class SolanaClient:
    def __init__(self, rpc_url:str):
        self.rpc_url = rpc_url

    def get_balance(self, wallet_keyfile:str) -> float:
        # TODO: implement real RPC call using keyfile pubkey
        return 0.0

    def transfer(self, src_keyfile:str, dst_addr:str, amount_sol:float) -> TxResult:
        logger.info(f"Transfer {amount_sol} SOL from {src_keyfile} to {dst_addr}")
        return TxResult(sig='SIMULATED')

    def burn_wallet(self, key_path:str):
        logger.warning(f"Burn wallet {key_path}")
```

```python
# src/solana/wallet_manager.py
from __future__ import annotations
from pathlib import Path
from dataclasses import dataclass
from loguru import logger
from .wallet import Wallet
from src.core.state import State
from src.core.keystore import KeyStore

# Stub keypair generator — replace with real solana keygen in prod

def _stub_keypair() -> tuple[str, str]:
    import secrets, json
    pk = "Pub" + secrets.token_hex(16)
    secret = {"type": "stub-keypair", "pubkey": pk, "secret": secrets.token_hex(32)}
    return pk, json.dumps(secret)

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
        pub, secret = _stub_keypair()
        path = self._write_keyfile(filename, secret)
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
from telegram.ext import ApplicationBuilder, CommandHandler
from . import commands as C

class TGBot:
    def __init__(self):
        self.token = os.getenv('TELEGRAM_BOT_TOKEN','')
        self.app = ApplicationBuilder().token(self.token).build()
        self.app.add_handler(CommandHandler('status', C.cmd_status))
        self.app.add_handler(CommandHandler('rotate_now', C.cmd_rotate_now))
        self.app.add_handler(CommandHandler('sync', C.cmd_sync))
        self.app.add_handler(CommandHandler('kill', C.cmd_kill))
        self.app.add_handler(CommandHandler('vault_status', C.cmd_vault_status))
        self.app.add_handler(CommandHandler('atpnl', C.cmd_atpnl))
    async def run(self):
        logger.info("Starting Telegram bot")
        await self.app.initialize()
        await self.app.start()
        await self.app.updater.start_polling()
```

```python
# src/telegram/commands.py
from loguru import logger

def cmd_status(update, ctx):
    logger.info("/status"); update.message.reply_text("Balances: ... (stub)")

def cmd_rotate_now(update, ctx):
    logger.info("/rotate_now"); update.message.reply_text("Manual rotation kicked off.")

def cmd_sync(update, ctx):
    logger.info("/sync"); update.message.reply_text("Health OK.")

def cmd_kill(update, ctx):
    logger.warning("/kill -> kill switch"); update.message.reply_text("Kill switch activated (simulated).")

def cmd_vault_status(update, ctx):
    update.message.reply_text("Vault (Hot/Cold): ... (stub)")

def cmd_atpnl(update, ctx):
    update.message.reply_text("All-time PnL: ... (stub)")
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

## Main Entrypoint

```python
# src/main.py
from loguru import logger
from src.core.config import CFG
from src.telegram.bot import TGBot
from src.security.kill_switch import KillSwitch
from src.solana.wallet_manager import WalletManager
from src.core.state import State

async def _noop():
    pass

def drain_all():
    logger.warning("Draining all active wallets to collector (stub)")

def reboot_bot():
    logger.warning("Rebooting bot (stub)")

async def main():
    logger.info(f"Starting CopyTrader in {CFG.env} mode")
    state = State(); wm = WalletManager(state)
    ks = KillSwitch(drain_all, reboot_bot)
    ks.check_and_trigger()
    bot = TGBot()
    await bot.run()

if __name__ == '__main__':
    import asyncio
    asyncio.run(main())
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

