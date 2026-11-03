# HEXORA

Website security assessment and client-side source collection for Termux and desktop. Python 3.10+.

**Use only on websites you own or have written permission to test.** HEXORA asks you to confirm this before every target.

## Install

Termux (Android):

```bash
pkg update && pkg install python git openssl -y
cd HEXORA
pip install -r requirements.txt
cp .env.example .env
```

Linux / macOS / Windows:

```bash
cd HEXORA
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env                                  # Windows: copy .env.example .env
```

All dependencies are pure Python, so they install on Termux without compiling anything.

## Telegram setup

1. Open `@BotFather` in Telegram, send `/newbot`, copy the token into `TELEGRAM_BOT_TOKEN` in `.env`.
2. Send any message to your new bot (or add it to a group and send a message there).
3. Open `https://api.telegram.org/bot<TOKEN>/getUpdates` in a browser and copy `"chat":{"id": ...}` into `TELEGRAM_CHAT_ID`. Group IDs are negative.

Without these values HEXORA still works and saves everything locally; delivery is reported as `not_configured`.

## Run

```bash
python main.py
```

Enter the URL, type `YES` to confirm authorization, then pick an option. Non-interactive:

```bash
python main.py --url https://example.com --authorized --option 5
```

| Option | What it does | Telegram gets |
|---|---|---|
| 01 | Full audit: runs 02-15 and 17-18, plus the source export | `{site}.json`, then `{site}.html` |
| 02-15 | One security module | `{site}_{module}.json` only |
| 16 | Get Source Code | `{site}.html` only |
| 17 | Phishing Check | `{site}_phishing.json` only |
| 18 | Spyware Check | `{site}_spyware.json` only |

Files are saved to `reports/` (JSON, plus `hexora.log`) and `source_exports/` (HTML). If the JSON upload fails in a full audit, the HTML is not sent and both files stay on disk.

## Phishing Check (17) and Spyware Check (18)

Both are static and non-executing: pages and in-scope scripts are fetched through the normal scope-checked client and parsed, nothing is run, no form is submitted and no credentials are ever sent. Out-of-scope third-party scripts and destinations are listed but not fetched; external hosts are only resolved through DNS.

- Verdicts: `phishing_confirmed | probable | suspected`, `spyware_confirmed | probable | suspected`, `no_*_indicators_detected`, `inconclusive`. A login form, a CDN, a permission API, an analytics SDK or a cross-origin form alone never produces a phishing/spyware verdict.
- Confirmation needs a verified threat-intelligence listing (phishing) or a listing plus a statically correlated sensitive-data flow (spyware). Without runtime observation, `permission_granted_observed` and `outbound_transmission_observed` are never reported.
- Data flows say whether a destination is `source_reference`, `statically_inferred` or `unresolved_dynamic`. An endpoint operator is never presented as the data recipient; developer identity is reported as unknown unless the page declares one.
- Results are in `modules[].data` of the JSON report (`verdict`, `confidence`, `risk_level`, `data_flows`, `destinations`, `network_observations`, `threat_intelligence`, `evidence`, `tests_performed`, `tests_not_performed`, `limitations`, `recommendations`, ...).
- Threat intelligence (optional): `URLHAUS_AUTH_KEY` (free abuse.ch key) and `PHISHTANK_APP_KEY`. Private/local targets are never sent to third-party services. A failed or unconfigured lookup is recorded as a limitation, never as a clean result. Disable with `HEXORA_THREAT_INTEL_ENABLED=false`. `HEXORA_KIT_SIGNATURES` can point to your own JSON list of named kit/family signatures.

## Authorized scope

The URL's host is in scope, plus its `www.`/apex twin. Add more with `HEXORA_EXTRA_HOSTS=api.example.com` or allow all subdomains with `HEXORA_ALLOW_SUBDOMAINS=true`. Every request and every redirect hop is checked against scope, and the destination IP is resolved and checked each time. Private, loopback, link-local and cloud-metadata addresses are blocked. `HEXORA_ALLOW_PRIVATE_TARGETS=true` allows private ranges for a lab you own; metadata endpoints stay blocked.

## Reading results

Severity: `critical` > `high` > `medium` > `low` > `informational`.

Status says how sure HEXORA is:

- `confirmed`: directly observed (for example a header is absent, a file is served).
- `probable`: test behavior strongly suggests the flaw, but it was not exploited (injection findings never go beyond this).
- `potential`: a risk indicator or version match that needs manual verification.
- `inconclusive`: the test could not decide.

Confidence (`high/medium/low`) is how reliable the evidence is. A missing header is a hardening gap, not proof of an exploit. The report always lists tests that were not performed; read that section before treating a clean result as safe.

Scan status: `completed` (no findings from the tests that ran), `completed_with_findings`, `partial` (a module failed, was cut short, or the crawl hit its limits), `failed`, `cancelled`.

## What each module really does

See `modules/`. In short: injection uses benign markers, a single quote and `{{7*7}}` on GET parameters only. Fingerprinting uses banners, cookies and HTML signatures. CVE correlation queries the NVD API (cached 24h) and reports version-based matches as `potential`. TLS uses a handshake, certificate expiry and legacy-protocol checks. Authorization checks only public admin-style paths. SSRF only verifies when you provide `HEXORA_SSRF_CALLBACK_URL`. The availability module sends 12 sequential requests and is not a load test.

## Known limitations

- No authenticated testing: IDOR/BOLA, password reset and role checks are not performed and are reported as such.
- Injection tests cover GET parameters found by a bounded crawl. No POST, blind or time-based tests, no OS command execution.
- Source collection is static: no JavaScript is executed and there is no headless browser, so resources loaded only at runtime are not found. A single HTML file cannot reproduce server-side behavior, and assets on other hosts are listed, not embedded. Backend code is never retrieved.
- DNS is checked before each request, but `requests` resolves again when it connects, so a DNS-rebinding race is reduced, not eliminated. Behind a proxy that resolves names itself, the local check is only advisory.
- The bundled dependency advisories are a short list. CVE data needs internet access to `services.nvd.nist.gov`.
- TLS legacy-protocol tests depend on the local OpenSSL build; when it cannot test them, that is reported.
- Concurrency uses a small thread pool, not asyncio.

## Troubleshooting

- `Rejected: ... private network address`: the target resolves to a private IP. Use a public target or set `HEXORA_ALLOW_PRIVATE_TARGETS=true` for your own lab.
- Many `failed` requests: raise `HEXORA_REQUEST_TIMEOUT`, or check that the site allows your IP.
- Telegram `401`: wrong token. `400 chat not found`: wrong chat ID, or you never messaged the bot.
- Telegram `not_configured`: `.env` still has the placeholder values.
- NVD lookups fail: the module reports `partial`; set `NVD_API_KEY` to avoid rate limits.

## Tests

```bash
python -m pytest -q
```

The suite runs against a local fixture site on 127.0.0.1 and mocks Telegram and NVD. It never touches real targets.
