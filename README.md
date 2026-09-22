# hjs - stealth headless browser with JS execution (Mojo + QuickJS)

Sibling of **hbrowser** / previous **hjs** zip. This build adds a
scraping/stealth toolkit on top of the JS engine:

## New in the stealth build

### Browser fingerprint profiles
`--profile=NAME` loads a full identity: User-Agent, `sec-ch-ua` client
hints, `Sec-Fetch-*` headers, `Accept` and `Accept-Language`, plus a
browser-matching TLS cipher list. Built-in profiles:

| profile | browser |
|---------|---------|
| `chrome131` | Chrome 131 desktop (Windows) |
| `chrome131-mobile` | Chrome 131 Android (Pixel 8) |
| `firefox133` | Firefox 133 desktop |
| `safari17` | Safari 17.6 (macOS) |
| `edge131` | Edge 131 desktop |

Profiles live in `browser_profiles.txt` (override path with
`HJS_PROFILES` env var). Add your own by copying a block.

### Cookies
- `--cookies=FILE` - read Netscape-format cookies before the run
- `--cookie-jar=FILE` - write all cookies after the run
Same jar file for both = persistent sessions across invocations, exactly
like `curl -b/-c`. Verified against httpbin.org.

### Anti-ban knobs
- `--referer=URL` - set Referer (also gets auto-set on JS sub-fetches)
- `--proxy=URL` - HTTP/HTTPS/SOCKS5 proxy (`socks5://host:1080`)
- `--http1` - force HTTP/1.1 (default tries HTTP/2 with fallback)
- `--min-tls=1.2|1.3` - TLS floor (default 1.2; 1.3 = shorter handshake)
- `--ciphers=LIST` - custom OpenSSL cipher list (profiles set this too)
- `--delay-ms=N` - politeness delay before every request
- `--retries=N` - auto-retry on 408/425/429/5xx (default 2)
- `--backoff-ms=N` - linear backoff base between retries (default 500)

Stalled transfers abort automatically (<1 KB/s for 30 s) so dead
connections never hang a scraping job.

### Captcha / bot-wall detection
Every response is scanned for Cloudflare (`cf-challenge`), PerimeterX,
reCAPTCHA, hCaptcha, DataDome, Incapsula. When detected the JSON output
gains a `"captcha": "cloudflare"` field - your scraper sees it without
parsing the body and can route the URL to a solver or flag it.

Example:
```json
{
  "url": "https://target.com/page",
  "status": 403,
  "captcha": "cloudflare",
  "attempts": 3,
  ...
}
```

## Usage

```bash
export LD_LIBRARY_PATH=/root/hjs:/root/hbrowser/mojo-home/lib

# Everything on: Chrome 131 identity, cookie jar session, politeness,
# retries
hjs https://target.com \
  --profile=chrome131 \
  --cookies=/root/hjs/cookies.txt --cookie-jar=/root/hjs/cookies.txt \
  --delay-ms=1500 --retries=3 \
  --links --mode=json
```

All previous hjs/hbrowser flags still work (`--mode=text|html|json`,
`--js/--no-js`, `--js-budget-ms`, `--timeout`, `--max`, `--header`,
`--ua`, `--links`, `--no-meta`).

## Files

- `hjs` - Mojo binary (150 KB)
- `libhjs.so` - QuickJS 2024-01-13 + C shim (3.8 MB)
- `hjs.mojo` - engine source
- `hjs_shim.c` - C shim source
- `browser_profiles.txt` - fingerprint profiles (editable text)
- `test_hjs3.py` - JS engine tests (fetch/timer/XHR)
- `test_stealth.py` - stealth feature tests (profiles/cookies/proxy)

## Honest limitations (read before relying on it)

- TLS fingerprinting: hjs sends browser-like cipher lists and HTTP/2, but
  the underlying TLS stack is OpenSSL (libcurl), so a JA3/JA4 hash will
  NOT match real Chrome. Sites doing TLS-level fingerprinting (rare, but
  some anti-bot vendors do) will still classify it as non-browser. For
  those targets you need `curl-impersonate` or a real browser.
- No cookie **consent-page interaction**, no CAPTCHA solving - detection
  only, so your orchestration can react.
- External `<script src>` files are not executed; inline scripts only.
