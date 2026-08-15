# Odds Scanner — Manual

Reference for running and configuring the paper-trading scanner.

Version 2.0.0 · Windows

---

## Contents

- [Everyday use](#everyday-use)
- [All commands](#all-commands)
- [Configuration](#configuration)
- [How it decides what to bet](#how-it-decides-what-to-bet)
- [Settlement](#settlement)
- [The spreadsheet](#the-spreadsheet)
- [Automation](#automation)
- [When something breaks](#when-something-breaks)
- [Reading your results](#reading-your-results)

---

# Everyday use

Every command starts here:

```powershell
cd $HOME\Documents\odds-scanner
```

**See where you stand:**
```powershell
python scanner.py --status
```

**Run a scan** (settles finished bets, then looks for new ones):
```powershell
python scanner.py
```

That's the whole daily loop. The scheduled task runs the second command every 4 hours by itself, so manual runs are only for when you want to check something now.

**Two habits that matter:**

Close Excel when you're done looking at the spreadsheet. Windows locks the file while it's open, and every scheduled run will fail until you close it.

Deal with anything under **NEEDS YOUR ATTENTION** in `--status`. Those are finished matches the scanner couldn't grade — put `won`, `lost`, or `void` in column L.

---

# All commands

| Command | What it does |
|---|---|
| `python scanner.py` | Settle finished bets, then scan for new opportunities |
| `python scanner.py --status` | Capital, open positions, settled results, P&L |
| `python scanner.py --dry-run` | Scan and report findings, write nothing |
| `python scanner.py --settle-only` | Grade finished bets, skip scanning |
| `python scanner.py --check` | Verify the setup is intact |
| `python scanner.py --test-key` | Test the API key with one request |
| `python scanner.py --list-tournaments 12` | List tournament IDs for a sport |
| `python scanner.py --verify-settlement ID` | Show how a result is being read |
| `python scanner.py --reset` | Clear the ledger, start fresh (backs up first) |
| `python scanner.py --version` | Show the version |
| `python scanner.py --help` | List all flags |

**Sport IDs:** `11` basketball · `12` tennis · `13` MLB

`--reset` asks you to type `RESET` to confirm. Add `--yes` to skip that prompt (used by scripts; be careful with it).

---

# Configuration

Everything lives in `config.json`, edited in Notepad. Only `ODDSPAPI_KEY` and `TRACKED_TOURNAMENTS` are required — everything else has a sensible default and can be left out entirely.

A minimal working file:

```json
{
  "ODDSPAPI_KEY": "your-key-here",
  "TRACKED_TOURNAMENTS": {
    "Wimbledon": 2810,
    "Canadian Open": 2847
  },
  "DAILY_BUDGET": 8
}
```

## Essentials

| Setting | Default | Meaning |
|---|---|---|
| `ODDSPAPI_KEY` | — | Your API key from oddspapi.io |
| `TRACKED_TOURNAMENTS` | `{}` | Which tournaments to watch. Get IDs from `--list-tournaments` |
| `DAILY_BUDGET` | `8` | Max API requests per day. Free tier is 250/month ≈ 8/day |
| `STARTING_CAPITAL` | `5000` | Opening bankroll in £ |

## Position sizing

| Setting | Default | Meaning |
|---|---|---|
| `ARB_STAKE_PCT` | `0.10` | Arb stake as a share of capital |
| `ARB_STAKE_CAP` | `750` | Hard ceiling on any arb stake, in £ |
| `KELLY_FRACTION` | `0.50` | Fraction of full Kelly for value bets. Lower is safer |
| `EV_STAKE_CAP_PCT` | `0.05` | Max share of capital on any one value bet |
| `MAX_EXPOSURE_PCT` | `0.40` | Stop opening positions past this much capital at risk |
| `SLIPPAGE` | `0.005` | Haircut on odds, since top-of-book rarely fills in full |

## Edge validation

| Setting | Default | Meaning |
|---|---|---|
| `MIN_BOOKS` | `3` | Books needed quoting both sides before any consensus |
| `MIN_INDEPENDENT_GROUPS` | `3` | Distinct *owners* needed. Raising this is the strongest filter |
| `MAX_CREDIBLE_EV` | `15.0` | Reject value bets above this EV% as stale prices |
| `MAX_CREDIBLE_ARB` | `10.0` | Reject arbs above this gross % as stale prices |

## Timing

| Setting | Default | Meaning |
|---|---|---|
| `PRIORITY_WINDOW_HRS` | `6` | Hours before start where scanning is concentrated |
| `RESCAN_COOLDOWN_MIN` | `45` | Won't re-scan the same match within this many minutes |
| `SKIP_INSIDE_MIN` | `20` | Ignore matches starting sooner than this — too late to bet |
| `SETTLE_DELAY_HRS` | `3` | Wait this long after start before asking for a result |
| `SCAN_RESERVE` | `2` | Requests always held back so settling can't starve scanning |
| `MIN_REQUEST_INTERVAL` | `2.0` | Seconds between API calls. Raise if you hit rate limits |

## Two rules for editing config.json

**Commas between entries, but not after the last one.**

```json
{
  "ODDSPAPI_KEY": "abc",     ← comma
  "DAILY_BUDGET": 8          ← no comma, it's last
}
```

**Percentages are decimals.** `0.10` means 10%, not 10.

If you break the syntax, the scanner tells you which line.

---

# How it decides what to bet

## 1. Which matches to look at

Scanning is weighted toward the hours just before a match, when team news, late money and liquidity shifts move prices most. Matches within 20 minutes are skipped — too late to place anything.

## 2. Finding a fair price

Each bookmaker's two-way price is **de-vigged**: strip out their margin so the two sides sum to 100% rather than 105%. The median across books becomes the consensus fair probability.

## 3. Spotting an opportunity

**Arbitrage** — best price on each side, where `1/odds₁ + 1/odds₂ < 1`. Profit regardless of outcome.

**Value** — a single price that beats the fair consensus. Positive expectation over many bets, but any one can lose.

## 4. Validating it

This is where most candidates die, and that's the point. Six checks:

**Leave-one-out consensus.** Fair value is recomputed *excluding* the book being judged. Otherwise a book's own price drags the "fair" value toward itself and the edge is partly circular. This typically cuts a measured edge by half or more, and the smaller figure is the honest one.

**Independent ownership.** Ladbrokes, Coral and Betdaq are all Entain. Betfair, Paddy Power and Sky Bet are all Flutter. Three books from one group is one opinion wearing three hats. Needs 3+ genuinely independent operators.

**Plausibility cap.** Above 15% EV or a 10% arb, the price is stale or broken rather than generous. You would never get the bet on.

**Outlier detection.** A price both statistically and materially (7+ percentage points) away from the pack is treated as stale.

**Pinnacle cross-check.** Pinnacle runs ~2–3% margin and doesn't limit winners, making it the sharpest reference available. If its de-vigged price says no edge, the bet is rejected.

**Extreme probabilities.** Below 5% or above 95%, EV estimates are too noisy to act on.

Every rejection is logged with its reason in `scanner-log.txt`. Reading those is genuinely useful — it shows what the scanner declined and why.

## 5. Sizing it

**Arbs:** 10% of capital, capped at £750.

**Value bets:** half-Kelly, capped at 5% of capital. Full Kelly assumes your probability estimate is exactly right; it never is, so half-Kelly cuts variance sharply for a small cost in growth.

**Both:** nothing new opens past 40% of capital at risk.

When two value bets appear on opposite sides of the same match, only the one with the higher Kelly growth rate is taken. They're mutually exclusive — one outcome, so backing both is just paying vig twice.

---

# Settlement

Runs automatically at the start of every scan, on matches that finished 3+ hours ago.

It grades a bet by itself when the result is unambiguous: either the feed names the player, or there are exactly two outcomes it can map onto the two participants.

**When anything is unclear, it refuses to guess** and flags the row instead. That happens when a name matches both players, when the result wording is unfamiliar, or when your selection matches neither participant.

That's deliberate. A wrongly settled bet quietly corrupts every number downstream and you'd have no reason to check. An unsettled one is obvious and takes seconds to fix.

Column **AC** records how each row was graded, so you can audit it later.

To settle manually: open the spreadsheet, put `won`, `lost`, or `void` in column L.

To check the grading is reading your feed correctly:

```powershell
python scanner.py --verify-settlement id1200828573252814
```

---

# The spreadsheet

`paper-trading-ledger.xlsx`, four tabs:

**Dashboard** — capital, P&L, costs, strike rate, drawdown, average edge captured.

**Assumptions** — the sizing and cost figures the spreadsheet's own formulas use. Note these are separate from `config.json`: the spreadsheet uses these for display, the scanner uses `config.json` for decisions. Keep them aligned if you change either.

**Trade Blotter** — one row per position. Blue columns are inputs, grey are calculated. Includes scenario columns showing what each open position pays if it wins or loses.

**Fee Reference** — every platform's commission structure.

You only ever type in **column L** (the result). Everything else is written by the scanner or calculated.

---

# Automation

The scheduled task runs every 4 hours.

**Check it's alive:**
```powershell
Get-ScheduledTask -TaskName "Odds Scanner" | Get-ScheduledTaskInfo | Select LastRunTime, LastTaskResult, NextRunTime
```

`LastTaskResult` of `0` means success.

**Pause it:**
```powershell
Disable-ScheduledTask -TaskName "Odds Scanner"
```

**Resume:**
```powershell
Enable-ScheduledTask -TaskName "Odds Scanner"
```

**Remove:**
```powershell
Unregister-ScheduledTask -TaskName "Odds Scanner" -Confirm:$false
```

**What must be true for a scheduled run to work:** PC on and not asleep, you logged in, Excel not holding the spreadsheet open, internet available. Nothing needs to be *open* — no PowerShell, no Claude, no browser.

Every run appends to `scanner-log.txt`.

---

# When something breaks

| Message | Cause and fix |
|---|---|
| `Permission denied: ...xlsx` | Excel has the file open. Close it |
| `No API key found` | Key missing from `config.json` |
| `config.json has a syntax error on line N` | Missing comma, quote or bracket |
| `No tournaments configured yet` | Run `--list-tournaments 12`, add IDs to config |
| `HTTP 401` / `403` | Key wrong, expired, or quota spent. Run `--test-key` |
| `HTTP 429` | Rate limited. Retries automatically; raise `MIN_REQUEST_INTERVAL` if persistent |
| `nothing in the priority window` | Normal. No matches within 12 hours |
| `thin market (2 books)` | Normal. Too few books to compare |
| `edge rejected (...)` | Working as intended. The validation caught an artifact |
| `NEEDS REVIEW` | Fill in column L yourself |
| `blotter is full` | 100 rows used. Archive and reset |
| `'python' is not recognized` | Python not on PATH. Reinstall with the PATH box ticked |

**When stuck, start here:**
```powershell
python scanner.py --check
```

---

# Reading your results

## What to watch

**Average edge captured (bps)** on the Dashboard is the most informative number. If it stays positive while net P&L doesn't follow over dozens of bets, the fair-price consensus is miscalibrated — the scanner is finding edge that isn't there.

**Costs as % of gross P&L** shows how much commission is eating. If it's above roughly a third, you're trading too often on exchanges for the edge you're capturing.

**Max drawdown** tells you whether your sizing is too aggressive. Persistent large drawdowns mean lowering `KELLY_FRACTION`.

## What to ignore

**Short-run P&L.** With fewer than 30 settled bets, the figure is noise. Winning 60% of 15 bets is entirely normal with no edge at all; so is losing 60%.

**Strike rate on its own.** Value betting at long odds should have a low strike rate and still be profitable. The number only means something alongside the odds you took.

## Verify the grading once

Pick a settled row tagged `[auto]` in `--status`, look up who actually won that match, and confirm the spreadsheet agrees.

This is the one part of the system never tested against live data. A grading error would be invisible in the numbers and would corrupt everything downstream. Worth ten minutes now rather than discovering it after fifty bets.

## The honest frame

The first month answers whether the machinery works: are prices read correctly, does settlement grade accurately, do the fee calculations match a real account. Those questions are answerable in weeks.

Whether there's real money in it needs hundreds of bets, and the answer may well be no. Arbitrage windows close fast, and +EV measured against a consensus is only as good as that consensus. The ledger exists to find that out cheaply, before any real money is at stake.

One cost this models nowhere: bookmakers limit or close accounts that win consistently, often within weeks. In practice that ends more arbitrage operations than commission ever does.
