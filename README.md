# WHOP AUTOMATION v2.0

> High-performance, anti-fingerprint automated checkout engine with BasisTheory tokenization, TLS browser impersonation, and a live zero-flicker ANSI dashboard.

[![Version](https://img.shields.io/badge/version-2.0-blue.svg?style=for-the-badge)](https://github.com)
[![Python](https://img.shields.io/badge/python-3.10%2B-blue.svg?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![License](https://img.shields.io/badge/license-MIT-green.svg?style=for-the-badge)](https://opensource.org/license/mit)
[![TLS](https://img.shields.io/badge/TLS-curl__cffi%20impersonation-orange.svg?style=for-the-badge)](https://github.com/lexiforest/curl_cffi)
[![TUI](https://img.shields.io/badge/TUI-zero--flicker%20ANSI-brightgreen.svg?style=for-the-badge)](core/terminal_ui.py)

---

## 1. Visual & Demo

![Demo](assets/demo.gif)

The dashboard is rendered with pure ANSI sequences via `core/terminal_ui.py` (`\x1b[H`, `\x1b[J`, `\x1b[K`), hiding/restoring the cursor and updating state in real time without flickering. Terminal preview (simulated to match the actual render layout):

```text
WHOP AUTOMATION v2.0                                          ● RUNNING  00:04:18

IDENTITY
  Account            user_gen8492@gmail.com
  Target             https://whop.com/checkout/plan_xxxxxxxxxxxx/
  Plan               Membership Access (plan_xxxxxxxxxxxx)
  Mode               CCN Bypass CVC
  Proxy Pool         12 / 12 alive

METRICS
  Progress           18 / 50  (36%)
  Hits               1
  Declines           16
  Decline Streak     2 / 5
  Retries            1
  Active Card        462239******0413  Visa Credit (US)
  Profile            mac_chrome

ACTIVITY
  [14:20:11]  ●  Processing 462239******0413
  [14:20:12]  ●  IP 198.51.100.24 | risk 0 | user_gen8492@gmail.com
  [14:20:16]  ✔  HIT 462239******0413 — 25.00 USD (Membership Access)
  [14:20:17]  …  Saved to data/output/whop_hits.txt
  [14:20:19]  ✖  Decline 462239******5678 — card_declined - Your card was declined. (streak 1/5)

[Ctrl+C to stop]
```

---

## 2. Purpose & Features

An automated checkout engine for payment flow research, infrastructure testing, and authorized security auditing. Built around speed, browser fingerprint camouflage, and proxy/token infrastructure management.

### Tokenization & Payment Orchestration
- Full Basis Theory Hosted Elements flow: session bootstrap, card container packaging, session key activation, independent expiry/CVC PATCH, and Confirmation Token extraction.
- Two card handling modes: **Mode 1 — CCN Bypass CVC** and **Mode 2 — Full CVV**.
- BIN-based card generation with proper Luhn checksum (`generate_cards_from_bin`, default 50 cards/BIN, duplicate protection via hash set).

### Anti-Fingerprint & Hardware Emulation
- TLS/JA3/JA4 and HTTP/2 fingerprint impersonation via `curl_cffi`: Chrome 124/131/136, Safari 17/18, Firefox 133.
- 18 hardware profiles (`core/hardware_profiles.py`): WebGL Vendor/Renderer, Canvas/Audio hash, Screen Resolution, Device Pixel Ratio, Hardware Concurrency, Sentry Tracing headers.
- Behavior telemetry v1 (`core/telemetry.py`): focus/paste/change/blur event chains with Gaussian distribution (`_rgauss`), session UUID, and WebRTC local IP.

### Proxy Management & Concurrency
- **Multi-threaded pre-flight health check**: `ThreadPoolExecutor` (default 20 workers) validating every proxy via `ip-api.com` — IP, latency, timezone, country code.
- **Geo-routing billing matcher**: offline matcher mapping proxy/BIN country to real billing addresses across 16 countries (US, GB, CA, AU, DE, FR, BR, VN, NL, IE, ES, IT, MX, JP, IN, RO) in `core/bin_matcher.py`.
- **Smart eviction**: dead proxies are dropped on TLS/timeout errors; the pool is only kept when more than one proxy remains.

### Resilience & Flow Control
- Per-attempt checkout/confirm data caching, isolating state between cards.
- Safe auto-stop after **5 consecutive declines** (decline streak), printing a session summary before exit.
- **Webhooks**: automatic HIT notifications via Discord Rich Embed and Telegram HTML (`core/notifier.py`).

---

## 3. Disclaimer & Security Warning

> **LEGAL WARNING & TERMS OF USE**
>
> This tool is developed for security research, educational purposes, and self-testing of payment infrastructure (authorized security auditing). The author accepts **no responsibility** for any misuse, financial damage, or **account suspension** resulting from violations of third-party Terms of Service. You use it entirely at your own risk.

### Information Security
1. **NEVER share** private tokens, cookies, session keys, webhook secrets, or proxy credentials in any form (screenshots, logs, issues, commits).
2. **Check `.gitignore`** before pushing — make sure local data files never reach the remote:

```gitignore
config.json
.env
data/input/proxies.txt
data/input/cards.txt
data/input/bins.txt
data/output/
data/last_run.json
__pycache__/
```

---

## 4. Installation & Usage (Quick Start)

### Requirements
- **Python 3.10+** (Windows / macOS / Linux — the codebase uses `str | None` type hints).
- An ANSI-capable terminal (Windows Terminal, iTerm2, VS Code Terminal). Legacy Windows CMD may render colors incorrectly — set `NO_COLOR=1` to disable styling.

### Step 1 — Install dependencies

```bash
pip install curl_cffi requests cloudscraper
```

| Package       | Role                                                                          |
|---------------|-------------------------------------------------------------------------------|
| `curl_cffi`   | **Required** — TLS/JA3/JA4 + HTTP/2 impersonation (Chrome/Safari/Firefox)     |
| `requests`    | Fallback when `curl_cffi` cannot be built in the environment                  |
| `cloudscraper`| Fallback for bypassing Cloudflare challenges when `curl_cffi` is unavailable  |

### Step 2 — Configure `config.json`

Create `config.json` in the project root (all keys are optional):

```json
{
  "preflight_proxy_check": true,
  "webhook": {
    "enabled": false,
    "discord_url": "",
    "telegram_bot_token": "",
    "telegram_chat_id": ""
  }
}
```

- `preflight_proxy_check: true` — enables the multi-threaded proxy health check before running (only applies when the pool has more than 1 proxy).
- `webhook.enabled: true` — enables HIT notifications; fill in `discord_url` and/or the `telegram_bot_token` + `telegram_chat_id` pair.

### Step 3 — Prepare input data

| File | Format | Example |
|------|--------|---------|
| `data/input/bins.txt` | `BIN\|MM\|YYYY` or `BIN\|MM\|YY` | `462239111654\|08\|2027` |
| `data/input/cards.txt` | `CC\|MM\|YYYY\|CVV` | `4622391116540413\|08\|2027\|031` |
| `data/input/proxies.txt` | `host:port:user:pass` or `http://user:pass@host:port` | `198.51.100.24:8080:user:pass` |

- Lines starting with `#` are treated as comments and skipped. Files support UTF-8 BOM.
- Mode 1: run from BINs (cards are generated with valid Luhn, you are asked how many cards per BIN, default 50). Mode 2: use existing cards from `cards.txt`.

### Step 4 — Launch

```bash
python main.py
```

The program prompts for: Checkout link → Card source (1/2) → Mode (1: CCN Bypass CVC, 2: Full CVV) → data file paths. Press **Ctrl+C** to stop safely (the UI restores the cursor).

---

## 5. Project Structure & Workflow

### Directory Tree

```text
whop/
├── core/
│   ├── bin_matcher.py        # Offline BIN matcher + billing profiles for 16 countries
│   ├── hardware_profiles.py  # 18 hardware profiles, WebGL/Canvas/Sentry trace
│   ├── notifier.py           # Discord Rich Embed & Telegram HTML webhooks
│   ├── proxy_manager.py      # Multi-threaded pre-flight check, ranking, eviction
│   ├── telemetry.py          # Browser behavior v1, Sardine UUID, WebRTC IP
│   └── terminal_ui.py        # Terminal UI v2.0 zero-flicker ANSI
├── data/
│   ├── input/
│   │   ├── bins.txt          # Source BIN list
│   │   ├── cards.txt         # Existing card list
│   │   └── proxies.txt       # Proxy pool
│   ├── output/
│   │   └── whop_hits.txt     # Successful payments (HITs)
│   └── last_run.json         # Last session state
├── config.json               # Proxy check & webhook configuration
├── main.py                   # Entry point & full flow orchestration
└── README.md
```

### Execution Flow (Input → Processing → Output)

```text
[data/input/]                  [PRE-FLIGHT]                          [PROCESSING]
bins.txt / cards.txt  ──────>  Proxy health check (20 workers)  ──┐
proxies.txt           ──────>  Rank by latency/country         ──┤
                                                                  ▼
                 Checkout link ──> Resolve Plan ID ──> Hardware profile + Telemetry
                                                                  │
                                                                  ▼
                     [OUTPUT]                       BasisTheory tokenize ──> Confirm token
  whop_hits.txt <── HIT record (card, amount, plan) <── 3DS / Status polling
  Discord / Telegram webhook <──────────────────────────┘
```

---

## 6. Troubleshooting / FAQ

**1. `ImportError: cannot import name 'requests' from 'curl_cffi'` / missing libraries**
Cause: `curl_cffi` is not installed or its binary is broken on your OS. Fix:
`pip install --upgrade curl_cffi`. On older Windows, install the Visual C++ Redistributable. If it cannot be built, the tool falls back to `cloudscraper` → `requests` (losing TLS impersonation, which is easier to block) — some requests may return 403 in that case.

**2. Proxies load but all fail the pre-flight check**
Cause: wrong format or dead proxies. Valid formats: `host:port:user:pass` or `http://user:pass@host:port` (a 2-part `host:port` also works). Files support UTF-8 BOM (the reader uses `utf-8-sig`). Verify a proxy is alive with `curl -x <proxy> https://api.ipify.org`.

**3. `Invalid checkout link!!!` or Plan ID cannot be resolved**
Cause: the link does not belong to `whop.com`, the short link (`ch_...`) has expired, or the checkout session has been reset. Fix: use a freshly generated direct checkout link, or enter the `plan_id` manually when the console asks. Responses containing `checkout session malformed` are retried up to 3 times, then that card is skipped (it does not count toward the decline streak).

**4. The tool stops after several cards: `5 consecutive declines — stopping`**
This is an intentional safety mechanism (decline streak ≥ 5) to avoid spamming and protect the account/IP. Fix: review BIN/card quality (prepaid cards decline more often), switch to a cleaner proxy range, or lower cards per BIN. Note: `3DS required` / `requires_action` also count toward the streak.

**5. HIT notifications are not sent to webhooks**
Verify `config.json`: `webhook.enabled` must be `true`; Discord needs a valid `discord_url` for the channel webhook; Telegram needs `telegram_bot_token` + `telegram_chat_id` (with the bot added to the group). Network errors while sending webhooks never interrupt the session — HITs are still written to `data/output/whop_hits.txt`.

---

## 7. Connect & Support

- **Discord**: [https://discord.gg/rxw8Hwk3qA](https://discord.gg/rxw8Hwk3qA)
- **TikTok**: [@lakofusion](https://www.tiktok.com/@lakofusion)
- **YouTube**: [@lakofusion2010](https://www.youtube.com/@lakofusion2010)

---

## 8. License

Released under the **MIT License**.

```text
MIT License

Copyright (c) 2026 lakofusion

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
