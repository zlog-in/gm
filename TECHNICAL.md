# GM (Generate Mnemonic) — Technical Specification

## 1. Overview

GM is a single-file, offline-only tool that deterministically derives BIP39 mnemonic phrases from user-provided data (text and/or files). It runs entirely in the browser with zero network requests and zero external dependencies.

**Core premise**: User provides memorable/personal input material → tool derives a deterministic cryptographic key → key is presented as a standard BIP39 mnemonic.

## 2. Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    index.html (single file)             │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Embedded Data                                   │   │
│  │  - BIP39 English wordlist    (2048 words)        │   │
│  │  - BIP39 Chinese wordlist    (2048 words)        │   │
│  │  - BIP39 Japanese wordlist   (2048 words)        │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │  UI Layer (HTML + CSS)                           │   │
│  │  - Text input (textarea)                         │   │
│  │  - File input (drag & drop / click)              │   │
│  │  - Parameter selection (word count, language)    │   │
│  │  - Result display (mnemonic grid, hex)           │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Crypto Layer (JavaScript, Web Crypto API)       │   │
│  │  - Input encoding & concatenation                │   │
│  │  - PBKDF2-SHA256 key derivation                  │   │
│  │  - BIP39 entropy-to-mnemonic conversion          │   │
│  │  - Self-verification test suite                  │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**File size**: ~42 KB (bulk is the three embedded wordlists).

## 3. Data Flow

```
User Input                  Encoding              Concatenation
─────────────────          ─────────              ─────────────
Text (any language)  ──►  TextEncoder (UTF-8)   ──┐
                                                  ├──►  inputBytes
File (any format)    ──►  FileReader (ArrayBuffer)┘

         │
         ▼

┌────────────────────────────────────────────-─┐
│  PBKDF2-SHA256                               │
│                                              │
│  Input:      inputBytes                      │
│  Salt:       <empty> (0 bytes)               │
│  Iterations: 1,000,000                       │
│  Output:     128 bits (12 words)             │
│              or 256 bits (24 words)          │
│                                              │
│  API: crypto.subtle.deriveBits()             │
└────────────────────────────────────────────-─┘
         │
         ▼  entropy (raw bytes)

┌────────────────────────────────────────────-─┐
│  BIP39 Conversion                            │
│                                              │
│  1. SHA-256(entropy) → checksum_hash         │
│  2. Take first N bits of checksum_hash       │
│     - 128-bit entropy → 4-bit checksum       │
│     - 256-bit entropy → 8-bit checksum       │
│  3. Concatenate: entropy_bits + checksum_bits│
│     - 128 + 4 = 132 bits                     │
│     - 256 + 8 = 264 bits                     │
│  4. Split into 11-bit segments               │
│     - 132 / 11 = 12 words                    │
│     - 264 / 11 = 24 words                    │
│  5. Each 11-bit value (0–2047) → wordlist    │
│                                              │
│  API: crypto.subtle.digest("SHA-256", ...)   │
└─────────────────────────────────────────────-┘
         │
         ▼

Output: mnemonic words + raw entropy hex
```

## 4. Input Processing

### 4.1 Text Input

- Encoded via `TextEncoder.encode()` — always produces UTF-8 bytes
- Empty string produces a 0-length Uint8Array (will be rejected before hashing)

### 4.2 File Input

- Read via `FileReader.readAsArrayBuffer()` — raw binary, format-agnostic
- Stored as `Uint8Array` in memory

### 4.3 Concatenation Rule

When both text and file are provided:

```
inputBytes = textBytes || fileBytes
```

Concatenation order is **text first, file second**. This is a fixed convention — reversing the order would produce a different mnemonic.

**Implication**: "hello" + photo.jpg ≠ photo.jpg + "hello". Users must remember both the content AND which inputs they used.

### 4.4 Validation

The only validation: at least one input (text or file) must be non-empty. No other constraints.

## 5. Key Derivation: PBKDF2

### 5.1 Parameters

| Parameter  | Value                      |
|-----------|----------------------------|
| Algorithm | PBKDF2                     |
| Hash      | SHA-256                    |
| Salt      | Empty (0 bytes)            |
| Iterations| 1,000,000                  |
| Output    | 128 bits (12w) or 256 bits (24w) |

### 5.2 Why PBKDF2

- Available natively in all modern browsers via `crypto.subtle`
- No external library needed — critical for the zero-dependency requirement
- Well-studied, NIST-approved (SP 800-132)
- 1M iterations provides meaningful brute-force resistance

### 5.3 Why No Salt

Design choice: the tool's purpose is **deterministic** regeneration — same input must always yield the same output. Adding salt would require users to remember an additional secret, defeating the "memorable input" goal.

Trade-off: without salt, identical inputs from different users produce identical outputs. This is acceptable because:
- The input material (personal text, personal files) is naturally unique per user
- The threat model is "someone who doesn't know my input tries to guess it", not "attacker has a precomputed table of common inputs"

### 5.4 Implementation

```javascript
// Step 1: Import raw input as PBKDF2 key material
const keyMaterial = await crypto.subtle.importKey(
  "raw", inputBytes, "PBKDF2", false, ["deriveBits"]
);

// Step 2: Derive bits
const derivedBits = await crypto.subtle.deriveBits(
  {
    name: "PBKDF2",
    hash: "SHA-256",
    salt: new Uint8Array(0),    // empty salt
    iterations: 1000000
  },
  keyMaterial,
  entropyBits                    // 128 or 256
);
```

Both calls delegate to the browser's native crypto implementation (BoringSSL in Chromium, NSS in Firefox, etc.), not to JavaScript.

## 6. BIP39 Mnemonic Generation

### 6.1 Standard

Follows [BIP-0039](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) exactly.

### 6.2 Checksum Calculation

```javascript
// SHA-256 hash of the entropy bytes
const hashBuffer = await crypto.subtle.digest("SHA-256", entropy);
const hashArray = new Uint8Array(hashBuffer);

// Checksum = first (entropyBits / 32) bits of the hash
// 128-bit entropy → 4-bit checksum (first nibble of first byte)
// 256-bit entropy → 8-bit checksum (full first byte)
const checksumBits = entropyBits / 32;
const checksumStr = hashArray[0].toString(2).padStart(8, "0").slice(0, checksumBits);
```

### 6.3 Word Mapping

```
entropy (binary string) + checksum (binary string) = combined bits
Split combined bits into 11-bit chunks
Each chunk → integer index (0–2047) → wordlist[index]
```

### 6.4 Wordlists

Embedded directly from the [bitcoin/bips](https://github.com/bitcoin/bips/tree/master/bip-0039) repository:

| Language           | Source file           | Words |
|-------------------|-----------------------|-------|
| English           | english.txt           | 2048  |
| Chinese Simplified| chinese_simplified.txt| 2048  |
| Japanese          | japanese.txt          | 2048  |

Wordlists are stored as space-delimited strings and split at runtime. The language selection only affects which wordlist is used for the final index→word mapping — the underlying entropy is identical regardless of language choice.

## 7. Self-Verification

On every page load, the tool runs 6 automated cryptographic tests in the background and reports results to the browser console.

### 7.1 BIP39 Test Vectors (Tests 1–4)

From the [official trezor/python-mnemonic test vectors](https://github.com/trezor/python-mnemonic/blob/master/vectors.json):

| # | Entropy (hex)                      | Expected Mnemonic                                                                                  |
|---|------------------------------------|----------------------------------------------------------------------------------------------------|
| 1 | `00000000000000000000000000000000`  | abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about      |
| 2 | `7f7f7f7f7f7f7f7f7f7f7f7f7f7f7f7f` | legal winner thank year wave sausage worth useful legal winner thank yellow                         |
| 3 | `80808080808080808080808080808080`  | letter advice cage absurd amount doctor acoustic avoid letter advice cage above                      |
| 4 | `ffffffffffffffffffffffffffffffff`  | zoo zoo zoo zoo zoo zoo zoo zoo zoo zoo zoo wrong                                                  |

These tests verify the entire BIP39 pipeline: entropy → SHA-256 checksum → bit concatenation → 11-bit splitting → word lookup.

### 7.2 PBKDF2 Cross-Reference (Tests 5–6)

Reference values computed independently via Python `hashlib.pbkdf2_hmac`:

```python
# Test 5
hashlib.pbkdf2_hmac('sha256', b'hello', b'', 1000000, dklen=16)
# → 871c4e10b03dad33c24fc304c4de52ad

# Test 6
hashlib.pbkdf2_hmac('sha256', b'hello', b'', 1000000, dklen=32)
# → 871c4e10b03dad33c24fc304c4de52ad15552eef6fbe5f8b267848ee21331e31
```

These tests verify that the browser's PBKDF2 implementation with our exact parameters (empty salt, 1M iterations) produces identical output to an independent implementation.

### 7.3 How to Check

Open the browser developer console (F12 → Console) after loading the page. Expected output:

```
[GM] All 6 cryptographic tests passed
   PASS  BIP39 vector 1
   PASS  BIP39 vector 2
   PASS  BIP39 vector 3
   PASS  BIP39 vector 4
   PASS  PBKDF2 (hello, 1M iter, 128bit)
   PASS  PBKDF2 (hello, 1M iter, 256bit)
```

Note: PBKDF2 tests take a few seconds due to 1M iterations.

## 8. Security Considerations

### 8.1 What This Tool IS

- A **deterministic** converter: input → mnemonic, reproducible forever
- An offline tool with zero network activity
- A convenience wrapper around standard, auditable algorithms

### 8.2 What This Tool IS NOT

- A random mnemonic generator (use hardware wallets or `crypto.getRandomValues()` for that)
- A password manager
- A replacement for proper key storage

### 8.3 Threat Model

| Threat                                | Mitigation                                              |
|---------------------------------------|---------------------------------------------------------|
| Input guessing (brute force)          | 1M PBKDF2 iterations; ~seconds per guess on modern CPU |
| Network exfiltration                  | Zero network calls; works offline; user can verify via DevTools Network tab |
| Malicious code injection              | Single self-contained file; no external scripts/CDNs    |
| Side-channel on shared machine        | User responsibility; tool does not persist any data     |
| Identical inputs from different users | Accepted trade-off (see §5.3); inputs are naturally unique |

### 8.4 Entropy Quality Warning

The output mnemonic is only as strong as the input. If the user inputs "password123", the resulting mnemonic has effectively ~0 bits of security despite being 12 BIP39 words. The 1M iterations of PBKDF2 add computational cost but cannot create entropy that isn't there.

**Users must understand**: the mnemonic's security = the unpredictability of their input, NOT the word count.

### 8.5 No Salt Trade-off

Without salt, an attacker who suspects the tool was used could build a targeted dictionary of likely inputs (e.g., famous quotes, common file contents) and precompute their outputs. The 1M-iteration cost makes this expensive but not impossible for a motivated attacker with a small candidate set.

## 9. Browser Compatibility

All crypto operations use the **Web Crypto API** (`crypto.subtle`), supported in:

| Browser          | Minimum Version |
|-----------------|----------------|
| Chrome/Edge      | 37+            |
| Firefox          | 34+            |
| Safari           | 11+            |
| Chrome Android   | 37+            |
| Safari iOS       | 11+            |

`FileReader`, `TextEncoder`, and CSS features used are supported in all browsers that support Web Crypto.

## 10. File Structure

```
gm/
├── index.html        # The complete tool (HTML + CSS + JS + wordlists)
└── TECHNICAL.md      # This document
```

Everything is in `index.html`. No build step, no dependencies, no install.

## 11. Reproducibility Guarantee

Given identical:
1. Input text (byte-for-byte, including encoding)
2. Input file (byte-for-byte)
3. Input order (text first, file second)
4. Word count setting (12 or 24)

The output mnemonic will be identical across:
- Any browser that implements Web Crypto API correctly
- Any operating system
- Any point in time

The algorithms (PBKDF2-SHA256, SHA-256, BIP39) are deterministic standards with no platform-dependent behavior. The wordlists are fixed by the BIP39 specification.

Language selection does NOT affect the underlying entropy — it only changes which word table maps the indices. Switching language produces different words but the same hex output.
