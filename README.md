# Paper Trader — Telegram Signal Follower

Listens to a public Telegram channel (built and tested against **BULLBURRZ**'s
GOLD signal format), simulates taking every signal as a paper trade against
live XAU/USD prices, and gives you a live dashboard with the day's P&L and
trading metrics. **No real money, no real broker, no real orders — ever.**
Everything is a simulation stored in a local SQLite file.

## How it works

```
Telegram channel ──▶ signal_parser.py ──▶ paper_broker.py ──▶ storage.py (SQLite)
                                                 ▲                    │
                                                 │                    ▼
                                    Twelve Data price polling   FastAPI + WebSocket
                                                                       │
                                                                       ▼
                                                            static/dashboard.html
                                                         (TradingView-style chart)
```

1. **`telegram_listener.py`** logs into Telegram as *your* user account (via
   [Telethon](https://docs.telethon.dev)) and watches the channel for new
   messages — including a 50-message backfill on startup so you don't lose
   context if you restart mid-day.
2. **`signal_parser.py`** classifies each message: a new entry signal, a
   `TP{n} HIT`, an `SL HIT`, `ALL TP SMASHED`, or just commentary it ignores.
3. **`paper_broker.py`** is the actual trading engine — see "Design decisions"
   below for exactly how it fills orders and sizes positions.
4. **`main.py`** polls live gold prices, feeds them to the broker so pending
   zone-orders can fill, and pushes everything to the dashboard over a
   WebSocket in real time.
5. **`metrics.py`** computes win rate, profit factor, expectancy, average
   R-multiple, drawdown, and streaks from the day's closed trades.

## Setup

### 1. Install dependencies
```bash
pip install -r requirements.txt
```

### 2. Get Telegram API credentials
Go to <https://my.telegram.org> → *API development tools* → create an app.
You'll get an `api_id` and `api_hash`. This uses **your own Telegram account**
to read the public channel (a bot can't be added as admin to a channel you
don't own, so a bot can't receive its posts — a regular user account can just
view any public channel it's a member of).

### 3. Get a market data key
Sign up free at <https://twelvedata.com> — their free tier explicitly
supports `XAU/USD`, which is why it's the default provider here (most free
forex APIs only cover currency pairs, not gold).

### 4. Configure
```bash
cp .env.example .env
# then edit .env with your Telegram api_id/api_hash and Twelve Data key
```

### 5. First run
```bash
python main.py
```
The first run will prompt you in the terminal for your phone number and the
login code Telegram texts you (standard Telethon login flow). After that, a
`.session` file is saved locally and you won't be prompted again.

Open **http://localhost:8000** for the dashboard.

### Recommended: dry-run first
Set `DRY_RUN=true` in `.env` and run it for a bit before trusting it with
paper money. It'll parse and log every message (check `paper_trader.db` →
`raw_messages` table) without opening any positions, so you can confirm the
parser is reading the channel correctly.

## Design decisions (read this before trusting the P&L)

These are judgment calls I made where the channel's behavior was ambiguous.
They're all in one place so you can change any of them if you'd model it
differently.

- **Entry fill model**: signals are a *zone* (e.g. "4463 - 4457"), not a
  single price. Positions sit `pending` until the live price actually trades
  into that zone, then fill at the touch price — like a real limit order.
  If price has already gapped past the zone by the time we see the signal,
  we don't chase; we fill at the zone boundary (see `_fill_entry` in
  `paper_broker.py`).
- **Position sizing**: fixed-fractional risk. Each signal risks
  `RISK_PER_TRADE_PCT` (default 1%) of current equity, sized so a full
  stop-out loses exactly that amount. Change this in `.env`.
- **TP index mapping**: `TP1 HIT` / `TP2 HIT` map directly, in order, to the
  `Target` list from the original signal. Each TP hit closes
  `1 / (number of targets)` of the position. Verified against the real
  "TP1 → TP2 → ALL TP SMASHED" sequences from the channel screenshots (see
  `tests/test_broker.py`).
- **Linking updates to positions**: primarily via Telegram's `reply_to_msg_id`.
  Some updates arrive *forwarded* from a second "VIP SIGNALS" channel, which
  can lose that reply link — if it can't be resolved, we fall back to
  "the one open position right now," and log a warning if that's ambiguous
  (more than one position open). Since this channel runs one active GOLD
  signal at a time in every sample we saw, this is a safe default — but
  **check the logs** (`paper_broker` logger) if you ever see it warn about
  ambiguous matching, and look at the `raw_messages` table to see what
  actually came through.
- **Messages we deliberately ignore**: pure hype/recap posts like
  "+380 PIPS" or "100PIPS CAPTURED" don't reference a specific TP/SL level,
  so they're logged as informational only and never change a position. This
  is intentional — guessing at their meaning risks corrupting your P&L.

## Extending it

- **New channel / new format**: `signal_parser.py` is fully independent of
  Telegram — feed it any string. Add its patterns alongside the existing
  ones and extend `tests/test_parser.py` with real samples the same way this
  one was built.
- **Different market data provider**: reimplement `get_latest_price` /
  `get_intraday_candles` in `market_data.py`; nothing else touches it
  directly.
- **Multiple channels / multiple symbols at once**: the schema already
  supports it (`symbol` is a column, not hardcoded) — you'd just point a
  second `TelegramListener` at another chat and feed the same `handle_telegram_message`.

## Disclaimer

This is a **paper trading / signal-tracking tool for your own analysis**. It
places no real orders and isn't connected to any broker. It's not affiliated
with BULLBURRZ or any signal provider, and nothing here is financial advice —
it's a way to see, with real numbers, how a signal source has actually
performed before you'd ever consider risking real money on it.
# telegrampaperTrader
