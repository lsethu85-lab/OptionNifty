# NIFTY Options Terminal

A single-file, self-contained HTML options-trading decision-support terminal covering NIFTY, BANKNIFTY, FINNIFTY, MIDCPNIFTY, and SENSEX — no backend, no build step, no dependencies. Open `index.html` (or `NIFTY_Options_Terminal.html`) directly in a browser.

> ⚠️ Educational decision-support tool only — not investment advice. Options trading can result in loss of the entire premium paid. Nothing in this app guarantees profit.

---

## What's real vs. simulated

This started as a fully simulated demo and has been progressively wired up to real market data. Current state, by feature:

| Data | Source | Status |
|---|---|---|
| Spot price (all 5 indices) | Yahoo Finance (`^NSEI`, `^NSEBANK`, `NIFTY_FIN_SERVICE.NS`, `NIFTY_MID_SELECT.NS`, `^BSESN`) | 🟢 Live |
| 1-year daily OHLC history | Yahoo Finance chart API | 🟢 Live — every technical indicator (EMA, RSI, MACD, ADX, SuperTrend, pivots, etc.) runs on real price bars |
| India VIX | Yahoo Finance (`^INDIAVIX`) | 🟢 Live |
| ATM IV estimate | Blend of live VIX + realized volatility computed from real closes | 🟢 Live-derived |
| IV Rank / Percentile | Rolling realized-vol history from real closes | 🟢 Live-derived |
| Expiry dates (weekly/monthly) | Real exchange calendar (NSE last-Tuesday / BSE last-Thursday rule, SEBI Sep-2025 one-weekly-per-exchange) | 🟢 Calculated, holiday-adjusted only when NSE/Angel confirms |
| Option chain (OI/Volume/LTP) | NSE `option-chain-indices` API via public CORS proxy | 🟡 Best-effort — NSE frequently blocks proxy traffic; falls back to a Black-Scholes model priced off real spot + real IV, with OI/Volume estimated |
| Strike Selector LTP (4 recommended strikes only) | **Angel One SmartAPI** (your own broker account) | 🔵 Best-effort, opt-in — see [Angel One setup](#angel-one-smartapi-setup) below |
| FII/DII flow, sector rotation | — | 🟠 Simulated. NSE/NSDL publish this once daily; reliably scraping it needs a server-side job, which a static HTML file can't do |

Every card/section that shows model-estimated data is badged so you always know what you're looking at:
- 🔵 **ANGEL ONE LIVE** — real LTP from your connected broker account
- 🟢 **NSE LIVE** — matched a real row in NSE's live option chain
- 🟠 **MODEL / SIMULATED** — computed from real spot + real IV, but NSE/broker data wasn't reachable

---

## Smart Strike Selector

The Strike Selector recommends one strike each for **Buy CE**, **Buy PE**, **Sell CE**, **Sell PE**, scored on delta sweet-spot, OI/OI-change, and IV rank.

**Fixes applied:**
- **Full descriptive naming** — cards now read e.g. `NIFTY 21st Jul (Weekly) 24250 CE` instead of a bare strike number, using the real computed expiry (weekly for NIFTY/SENSEX, monthly-only for BANKNIFTY/FINNIFTY/MIDCPNIFTY per the Nov-2024 weekly discontinuation).
- **Deterministic strike selection** — the model's OI/volume fallback used to draw from a shared, continuously-advancing random stream, so the "best" strike could silently change between renders. It's now seeded per strike + symbol + trading day, so the same strike scores the same all day.
- **Snapshot-locked Add to Portfolio** — the exact numbers shown on a card are frozen the moment it renders; clicking "+ Add to Portfolio" always books that frozen price, never a value recomputed at click-time.
- **Signal banner** — a `Signal: BUY CE / BUY PE / WAIT` card at the top, combining trend (EMA/RSI/MACD/ADX), PCR, IV rank, and institutional flow into a directional bias with reasoning bullets.
- **Angel One LTP overlay** — once connected, the 4 recommended strikes get their premium replaced with your broker's real last-traded price (see below).

---

## Angel One SmartAPI setup

Angel One's SmartAPI is directly browser-callable (CORS-enabled), unlike NSE's option-chain API — no proxy needed. It doesn't expose a full option chain, so it's used narrowly: to fetch **real LTP for just the 4 strikes shown on the Strike Selector**.

### 1. Create a SmartAPI developer account
Go to **smartapi.angelone.in** → Sign Up. This is a separate developer login from your regular Angel One trading login.

### 2. Create an app to get your API Key
Profile → **Create an App** → choose **Trading APIs**.
- **App Name** — anything
- **Redirect URL** — any real HTTPS URL (e.g. your GitHub profile); this app logs in directly via `loginByPassword`, so the redirect is never actually used
- **Primary Static IP** — required as of Angel's SEBI-compliance rollout. Use your current public IP (check via `https://api.ipify.org`). Angel's docs note the static-IP check is enforced for Orders/GTT calls, not for the read-only market-data calls this app makes — but the form requires a value regardless.

Copy the generated **API Key**.

### 3. Set up TOTP
Go to **smartapi.angelone.in/enable-totp**, log in with your Client ID + trading MPIN, verify the OTP, and when the QR code appears, look for **"can't scan? enter this key manually"** — that reveals the base32 secret string (e.g. `JBSWY3DPEHPK3PXP`). That's what this app needs (it generates fresh 6-digit codes from it client-side).

### 4. Connect in the app
Settings → Angel One SmartAPI panel:

| Field | Value |
|---|---|
| API Key | From step 2 |
| Client Code | Your Angel One trading Client ID (e.g. `A123456`) — **not your email** |
| MPIN / Password | Your 4-digit trading MPIN |
| TOTP Secret | Base32 string from step 3 |

Click **Connect**. Status will show `CONNECTED` with your client code on success.

### Verifying it's actually working
Login succeeding and the strike-price overlay succeeding are two different things. Go to **Smart Strike Selector** and check the **"Angel One feed — last attempt"** diagnostic card at the bottom — it lists each of the 4 strikes with either a green ✓ and the real LTP, or an amber ✗ with the specific reason (symbol-format mismatch, no trades yet on that strike, network error, etc.). No need to open browser dev tools.

**Note:** outside NSE market hours (9:15 AM–3:30 PM IST, Mon–Fri) the quote API will often return `ltp=0` for illiquid strikes — that's expected, not a bug.

### Security notes
- MPIN and TOTP secret are kept **in memory only** for the browser tab — never written to `localStorage`, so you'll re-enter them after every page reload.
- API Key and Client Code are saved to `localStorage` for convenience.
- Credentials never leave your browser except to Angel One's own API — there's no backend server in this project.
- **Never hardcode your MPIN or TOTP secret into the file itself** for convenience, even in a private repo — private repos still get cloned, forked, or occasionally misconfigured to public.

---

## Other fixes in this round

- **Auto-refresh no longer wipes in-progress form input.** The periodic background refresh (default every 5s) used to fully re-render whichever tab was open, which would erase anything mid-typed into the Angel One credential fields before they were saved. The Settings tab is now exempt from that periodic re-render; direct navigation (tab clicks, Save, Connect) still renders normally.
- **Option Chain expiry header** now shows the real weekly/monthly date, reconciled against NSE's confirmed expiry when the live chain is reachable (accounts for holiday shifts the calendar model can't predict on its own).
- **Portfolio positions** display the full contract name (symbol + real expiry + strike + type + live/model marker) instead of just a strike number.

---

## Known limitations

- **NSE option chain** requires session cookies NSE's site sets after a browser visit; a proxy relaying a single anonymous request often gets blocked. When this happens, the app falls back to a Black-Scholes model using real spot + real IV — clearly badged, not silently faked.
- **Angel One SmartAPI** doesn't provide a full option chain endpoint, only per-instrument quotes/greeks — hence the narrow scope (4 strikes only, not the whole chain).
- **Expiry calendar** doesn't account for individual exchange holidays unless NSE or Angel One confirms the actual date live; treat the calendar-computed date as an estimate until confirmed.
- **FII/DII flow** is simulated — this would need a server-side scraper against NSE/NSDL's daily publications, which a static HTML file can't run.
- Public CORS proxies (allorigins.win, corsproxy.io, codetabs.com) are rate-limited and occasionally go down; if all three fail, the app gracefully degrades to simulated mode rather than breaking.

---

## File structure

Everything lives in one HTML file — no build step, no `node_modules`, no external config. Safe to commit to a private GitHub repo as-is (see security notes above on credentials).
