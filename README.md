# GM (Generate Mnemonic)

Deterministic BIP39 mnemonic generator. Turn memorable input into a standard mnemonic phrase — reproducible forever, stored nowhere.

**Use your life to define randomness.** Your seed comes from three dimensions you choose:

- **A meaningful text** — a personal story, a private quote, a passage only you know.
- **A meaningful file** — a photo, a video, a voice recording, a GIF, any format.
- **A meaningful number** — via custom iteration count. A date, a phone number, a number only you know.

Combine any or all. Each dimension an attacker must guess multiplies the search space exponentially. You don't memorize anything random — your memories ARE the key.

## How It Works

```
Your input (text / file / both) + iteration count
        │
        ▼
   PBKDF2-SHA256  (custom iterations, no salt)
        │
        ▼
   BIP39 mnemonic  (12 or 24 words)
```

Same input, same iterations, same output — on any device, any browser, any time.

## Usage

Open `index.html` in a browser. That's it.

- No install, no build, no dependencies, no network.
- Works offline. Disconnect your WiFi if you want.

### Steps

1. Enter text and/or upload a file.
2. Choose word count (12 / 24), language (EN / 中文 / 日本語), and iteration count.
3. Click **Generate Mnemonic**.
4. Copy the result or inspect the raw hex.

## Features

- **Zero dependencies** — single 42KB HTML file, everything embedded.
- **Offline-only** — no network requests, ever. Verify in DevTools Network tab.
- **Three BIP39 wordlists** — English, Chinese Simplified, Japanese (from [bitcoin/bips](https://github.com/bitcoin/bips/tree/master/bip-0039)).
- **Configurable iterations** — 100K / 500K / 1M (default) / 5M / custom.
- **Wordlist reference panel** — browse all 2048 words, with generated words highlighted.
- **Self-verifying** — runs 6 cryptographic tests on every page load (check browser console).

## Security Model

| Layer | Protection |
|-------|-----------|
| Input unpredictability | Your personal text/files are the entropy source |
| Computational cost | 1M PBKDF2 iterations = ~seconds per brute-force guess |
| Offline execution | No data leaves your browser |
| Standard algorithms | PBKDF2, SHA-256, BIP39 — audited, NIST-approved |
| Native crypto | Web Crypto API (BoringSSL / NSS), not hand-written JS |

**The mnemonic is only as strong as your input.** "password123" produces a valid-looking 12-word phrase with near-zero real security. Choose input that only you can reproduce.

## Self-Verification

Open browser console (F12) after loading. You should see:

```
[GM] All 6 cryptographic tests passed
   PASS  BIP39 vector 1
   PASS  BIP39 vector 2
   PASS  BIP39 vector 3
   PASS  BIP39 vector 4
   PASS  PBKDF2 (hello, 1M iter, 128bit)
   PASS  PBKDF2 (hello, 1M iter, 256bit)
```

Tests 1-4 use [official BIP39 test vectors](https://github.com/trezor/python-mnemonic/blob/master/vectors.json). Tests 5-6 cross-reference against Python `hashlib.pbkdf2_hmac`.

## Important Notes

- **Byte-exact**: "Hello" and "hello" produce different results. A trailing space matters. A re-compressed photo is a different file.
- **Order matters**: text + file ≠ file + text. Text is always concatenated first.
- **Iterations are part of the protocol**: generating with 1M and restoring with 500K gives a different mnemonic.
- **Iterations as implicit salt**: GM has no traditional salt, but a custom iteration count serves the same purpose — different iterations on the same input produce completely different outputs. Pick a number that means something to you (a date, a phone number, any integer from 1 to 100,000,000) and it becomes your "second key". An attacker must guess both your input **and** your iteration count.
- **Language only changes words, not entropy**: switching from English to Chinese produces different words but the same underlying hex.

## Files

```
gm/
├── index.html      The tool (HTML + CSS + JS + wordlists)
├── TECHNICAL.md    Technical specification
├── BLOG.md         Technical blog post (Chinese)
└── README.md       This file
```

## Browser Support

Chrome 37+ / Firefox 34+ / Safari 11+ / Edge 37+ (any browser with Web Crypto API).

## License

MIT
