# 1LIFE — One-Decrypt Encryption

> *One message. One chance. No second chances.*

**1LIFE** is a browser-based encryption tool built around the **ODE (One-Decrypt Encryption)** protocol — a novel cryptographic scheme where each ciphertext can only be decrypted **exactly once**. After the first successful decryption, the key material is permanently and irrevocably destroyed through three independent enforcement layers. No servers. No cloud. Pure mathematics.

---

## Table of Contents

- [Overview](#overview)
- [Encryption Modes](#encryption-modes)
- [ODE Protocol — How It Works](#ode-protocol--how-it-works)
- [Three-Layer Destruction System](#three-layer-destruction-system)
- [Blob File Format](#blob-file-format)
- [Cryptographic Algorithms](#cryptographic-algorithms)
- [Known Loopholes & Limitations](#known-loopholes--limitations)
- [Security Properties](#security-properties)
- [Usage Guide](#usage-guide)
- [Technical Stack](#technical-stack)
- [Browser Compatibility](#browser-compatibility)
- [Developer Notes](#developer-notes)
- [Credits](#credits)

---

## Overview

1LIFE implements the **ODE v1 standard** alongside four additional industry-grade cipher modes. The key innovation of ODE is that decryption is not just a read operation — it is a **destructive, irreversible state transition**. Once a blob is decrypted:

- The key derivation components (salt + nonce) are **overwritten with zeros**
- A SHA-256 **fingerprint** of the blob is saved to `localStorage`
- The **hash chain** is broken permanently
- The blob file is mutated on download

All three layers must be independently bypassed to decrypt twice — which is cryptographically and architecturally prevented.

---

## Encryption Modes

| Mode | Algorithm | Key Type | Decryption Limit | Use Case |
|------|-----------|----------|------------------|----------|
| **ODE · 1-Life** | AES-256-GCM + SHA3-512 hash chain | Auto-derived | **Once only** | Self-destruct secrets |
| **AES-256-GCM** | AES-GCM + PBKDF2 (200k iterations) | Password | Unlimited | Secure standard messaging |
| **ChaCha20-Poly1305** | ChaCha20 stream + PBKDF2 (100k iterations) | Password | Unlimited | Mobile / low-power devices |
| **AES-256-CBC** | AES-CBC + HMAC-SHA256 + PBKDF2 (150k iterations) | Password | Unlimited | Legacy compatibility |
| **Hybrid RSA+AES** | RSA-2048 + AES-256-GCM session key | Auto keypair | Unlimited | Asymmetric use, no password |

---

## ODE Protocol — How It Works

### Encrypt

```
salt  = CSPRNG(32 bytes)
nonce = CSPRNG(16 bytes)
K     = SHA3-256("ODE-KEY-DERIVATION-v1" || salt || nonce)
H     = SHA3-512("ODE-CHAIN-v1" || K || nonce || salt)
C, tag = AES-256-GCM(plaintext, K, nonce)
blob  = "ODE1" || salt || nonce || H || tag || C
```

The key `K` is **never stored** — only its derivation inputs (salt, nonce) are embedded. The hash chain `H` binds all three components. Destroying either salt or nonce makes key derivation impossible.

### Decrypt (one time only)

```
1. Compute fingerprint = SHA-256(blob[0:128])
   If fingerprint in localStorage → BLOCK (Layer 1)

2. If blob[4:36]  == 0x00 * 32 → BLOCK (Layer 2)
   If blob[36:52] == 0x00 * 16 → BLOCK (Layer 2)

3. Recover K = SHA3-256("ODE-KEY-DERIVATION-v1" || salt || nonce)
   expected  = SHA3-512("ODE-CHAIN-v1" || K || nonce || salt)
   If H != expected → TAMPERED

4. plaintext = AES-256-GCM-Decrypt(C, K, nonce, tag)

5. DESTROY:
   blob[4 : 4+32+16+64] = 0x00  (zero salt + nonce + chain)
   localStorage.add(fingerprint)

6. Return plaintext
```

---

## Three-Layer Destruction System

Each layer is **independently sufficient** to block re-decryption. An attacker must bypass all three simultaneously.

### Layer 1 — Fingerprint Blacklist (localStorage)

- A SHA-256 digest of the first 128 bytes of the original blob is computed **before destruction**
- Stored in `localStorage` under key `1life_v2_fps` as a JSON array
- **Survives**: page refresh, tab close, browser restart, re-upload of the original untouched file
- **Does not survive**: clearing browser storage, incognito mode, different browser/device

### Layer 2 — Zeroed Header

- Bytes `[4 : 116]` of the blob (salt + nonce + chain) are overwritten with `0x00`
- This is written to the **downloaded destroyed blob file**
- Any re-upload of the destroyed file fails immediately — before any cryptographic operation
- **Survives**: as long as the user downloads and uses the destroyed file

### Layer 3 — Hash Chain Verification

- SHA3-512 chain binds key + nonce + salt with domain-separated prefix
- Zeroing salt/nonce (Layer 2) also breaks the chain — double-enforcement
- Tampering any byte in the header produces a different chain and fails verification
- AES-256-GCM authentication tag provides additional integrity over the ciphertext

---

## Blob File Format

### ODE Blob (`.ode`)

```
Offset  Size   Field
──────────────────────────────────────────────
0       4      Magic: "ODE1" (0x4F 0x44 0x45 0x31)
4       32     Salt (random, CSPRNG)
36      16     Nonce (random, CSPRNG)
52      64     Hash Chain (SHA3-512)
116     16     AES-GCM Auth Tag
132     N      Ciphertext (N = plaintext length)

Total: 132 + N bytes
```

### Post-Destruction ODE Blob

```
Offset  Size   Field
──────────────────────────────────────────────
0       4      Magic: "ODE1" (unchanged)
4       112    ZEROED (salt + nonce + chain = all 0x00)
116     16     Auth Tag (unchanged)
132     N      Ciphertext (unchanged but undecryptable)
```

### Other Formats

| Format | Magic | Description |
|--------|-------|-------------|
| `.enc`  | `AESGCM` | AES-256-GCM password-based |
| `.cc20` | `CC20P1` | ChaCha20-Poly1305 |
| `.cbc`  | `AESCBC` | AES-256-CBC + HMAC |
| `.hyb`  | `HYBRID` | RSA-2048 + AES-256-GCM |

---

## Cryptographic Algorithms

### SHA-3 (Keccak-f[1600])

1LIFE implements SHA3-256 and SHA3-512 in **pure JavaScript** (no external library) for use in the browser. The implementation uses the standard Keccak permutation with:

- Rate: 1088 bits (SHA3-256) / 576 bits (SHA3-512)
- Capacity: 512 bits / 1024 bits
- Padding: `0x06` (SHA-3 domain separation, not original Keccak `0x01`)

### AES-256-GCM

Uses the **Web Crypto API** (`crypto.subtle`) for hardware-accelerated AES-GCM:
- 256-bit key, 128-bit authentication tag
- 96-bit IV (12 bytes) for standard GCM operation
- Authenticated encryption — any modification to ciphertext or header is detected

### PBKDF2

Password-based key derivation for non-ODE modes:

| Mode | Iterations | Hash |
|------|------------|------|
| AES-256-GCM | 200,000 | SHA-256 |
| ChaCha20 | 100,000 | SHA-256 |
| AES-256-CBC | 150,000 | SHA-256 |

### RSA-OAEP (Hybrid Mode)

- 2048-bit modulus, public exponent `65537`
- Hash: SHA-256
- One-time keypair generated per encryption session
- Private key exported as JWK and embedded in the blob (no separate key management)

---

## Known Loopholes & Limitations

> This section documents **undisclosed vulnerabilities** in the 1LIFE/ODE scheme. These are not hypothetical — they are real architectural gaps that a sufficiently motivated adversary can exploit.

---

### Loophole 1 — The localStorage Bypass (Cross-Device / Incognito)

**Severity: HIGH**

The fingerprint database lives in `localStorage` — which is **browser-local and non-persistent** under certain conditions.

**Exploitation paths:**
- Open the original `.ode` file in a **different browser** (Chrome → Firefox) — fingerprint database is not shared
- Open in an **incognito / private window** — localStorage is isolated and cleared on close
- Open on a **different device** entirely — Layer 1 does not exist
- Clear browser storage (`localStorage.clear()`) — all fingerprints erased
- Use a **browser automation tool** (Puppeteer, Playwright) with a fresh profile

**Impact:** A recipient who knows about this can decrypt the same ODE blob multiple times by simply rotating through browser contexts.

**Mitigation (not implemented):** A server-side key revocation oracle would close this gap — but defeats the "no servers" design goal.

---

### Loophole 2 — Pre-Decryption Blob Copy (File-Level Snapshot)

**Severity: HIGH**

The ODE destruction only modifies the **in-browser memory buffer** and the **downloaded destroyed blob**. The original file on disk is **never touched** by the browser (browsers cannot overwrite arbitrary files on the filesystem by design).

**Exploitation path:**
1. Receive a `.ode` file
2. Make a copy: `cp secret.ode secret_backup.ode`
3. Decrypt `secret.ode` via 1LIFE (blob is destroyed in memory, destroyed version downloaded)
4. **Ignore the destroyed blob** — never replace the original
5. Clear `localStorage`
6. Decrypt `secret_backup.ode` — succeeds

**Impact:** Any recipient who copies the file before decrypting (or simply doesn't download the destroyed blob) can decrypt unlimited times. The "destruction" only works if the recipient **cooperates**.

**Root cause:** Browsers have no authority over the host filesystem. The ODE scheme requires **trust in the recipient's compliance** — which is not a security property.

---

### Loophole 3 — Memory Snapshotting / Browser DevTools

**Severity: MEDIUM**

At the moment of decryption, the plaintext exists in **JavaScript memory** as a `Uint8Array`. This is readable via:

- **Browser DevTools** → Memory tab → heap snapshot
- **JavaScript injection** (if the page is not served over HTTPS)
- **OS-level process memory dump** of the browser process

The plaintext is also rendered directly into the DOM (`outTxt.textContent = display`) — it exists as a string in the DOM tree, accessible via `document.getElementById('dec-output-text').textContent` from any script with DOM access.

**Impact:** The plaintext can be extracted at display time regardless of the cryptographic strength of the outer layer.

---

### Loophole 4 — SHA-3 Pure JS vs Native Timing

**Severity: LOW–MEDIUM**

The SHA-3 implementation is written in **pure JavaScript** and does not use constant-time operations for the Keccak permutation. While this is unlikely to be exploitable in a browser context (no direct timing channel), it means:

- No protection against **timing side-channel attacks** in a controlled measurement environment
- Performance is significantly slower than native implementations — making brute-force detection of the key derivation harder to notice

**Note:** The Web Crypto API operations (AES-GCM, PBKDF2, RSA-OAEP) **are** constant-time as they use the browser's native crypto implementation.

---

### Loophole 5 — Hybrid Mode Private Key Exposure

**Severity: MEDIUM**

In Hybrid (RSA+AES) mode, the RSA private key is **exported as JWK and embedded directly in the blob**. This means:

- Anyone who receives the blob has the private key
- There is no asymmetric access control — encryption and decryption capability are identical
- The "asymmetric" label is misleading for the current implementation

**Proper asymmetric encryption** would require the sender to have the recipient's public key and never embed the private key. The current implementation is effectively symmetric — just with RSA-wrapped key material.

---

### Loophole 6 — No Forward Secrecy Between Sessions

**Severity: LOW**

ODE derives the key deterministically from salt + nonce. If an attacker:
1. Captures the blob **before** decryption (network interception, file access)
2. Obtains the salt and nonce (which are in the blob in plaintext)

They can recompute the key and decrypt without triggering any of the three destruction layers — because destruction only runs **during a decryption attempt through the 1LIFE interface**.

The key material embedded in the unmodified blob is permanently sufficient to reconstruct `K`. An attacker with direct access to the blob file **never needs to go through the 1LIFE interface**.

---

### Loophole 7 — localStorage Poisoning / Fingerprint Collision

**Severity: LOW**

The fingerprint is SHA-256 of the first 128 bytes. Two crafted blobs with the same first 128 bytes would have the same fingerprint. This is a **second-preimage resistance** assumption on SHA-256 — considered safe today but worth noting.

Additionally, if an attacker can **write to localStorage** (XSS vulnerability in any page on the same origin), they could either:
- Pre-populate a fingerprint to block legitimate decryption
- Clear the fingerprint database to re-enable decryption

---

### Summary Table

| # | Loophole | Severity | Bypasses Layers | Fixable Without Server? |
|---|----------|----------|-----------------|------------------------|
| 1 | Cross-device / incognito localStorage | HIGH | Layer 1 | No |
| 2 | Pre-decryption file copy | HIGH | All 3 | No |
| 3 | Memory / DOM snapshotting | MEDIUM | N/A (post-decrypt) | Partial |
| 4 | Pure JS SHA-3 timing | LOW | None | Yes (use SubtleCrypto) |
| 5 | Hybrid mode private key embedding | MEDIUM | N/A (design flaw) | Yes (proper PKI) |
| 6 | Direct blob key recovery | LOW | All 3 | No |
| 7 | localStorage fingerprint collision/poisoning | LOW | Layer 1 | Partial |

---

### Bottom Line

**ODE enforces one-decrypt only against a *cooperative* recipient using the same browser.** It is not a security guarantee against an adversary with direct file access or alternate browser contexts. The scheme is best understood as a **social/UX enforcement mechanism** with cryptographic underpinning — not a hardware-enforced or server-backed guarantee.

For true one-time decryption with adversarial enforcement, consider:
- Hardware Security Modules (HSM) with key deletion policies
- Server-side key escrow with audit log and one-time key release
- Quantum key distribution (QKD) for physical one-time use

---

## Security Properties

| Property | Status | Notes |
|----------|--------|-------|
| Confidentiality | ✅ Strong | AES-256-GCM is quantum-resistant to classical attacks |
| Integrity | ✅ Strong | AES-GCM auth tag + SHA3-512 chain |
| Authenticity | ⚠️ Partial | No digital signature — cannot verify sender identity |
| One-decrypt (cooperative) | ✅ Yes | Three layers enforce this when recipient uses 1LIFE |
| One-decrypt (adversarial) | ❌ No | File copy + alternate browser defeats all layers |
| Forward secrecy | ❌ No | Key derivable from blob indefinitely |
| Post-quantum resistance | ⚠️ Partial | AES-256 is safe; SHA-3 is safe; RSA-2048 is not PQ-safe |
| Zero-knowledge | ❌ No | Plaintext visible in DOM during decryption |
| Server-free operation | ✅ Yes | Entirely client-side |

---

## Usage Guide

### Encrypt a Message (ODE Mode)

1. Open `1life.html` in any modern browser
2. Select **ODE · 1-Life** as the encryption mode
3. Type or paste your message
4. Click **Encrypt** (or press `Ctrl+Enter`)
5. **Download** the `.ode` file — this is your sealed blob
6. Share the `.ode` file with the intended recipient

> No password required for ODE mode. The key is auto-derived and self-contained.

### Decrypt an ODE Blob

1. Open `1life.html` in a browser
2. Switch to the **Decrypt** tab
3. Upload the `.ode` file
4. The inspect panel shows blob status — verify it reads `INTACT`
5. Click **Decrypt**
6. **Download the destroyed blob** if you need to prove the key was used
7. The original file's fingerprint is now saved — it cannot be decrypted again on this device

### Password-Based Modes (AES-GCM, ChaCha20, AES-CBC)

1. Select the desired cipher mode
2. Enter your message and a **strong password**
3. Encrypt and share the file + password separately (never together)
4. Recipient uploads the file, enters the password, decrypts

### Hybrid RSA+AES Mode

1. Select **Hybrid RSA+AES**
2. No password required — a one-time RSA-2048 keypair is generated automatically
3. Encrypt and download the `.hyb` file
4. Recipient uploads the file and decrypts without a password (private key is embedded)

---

## Technical Stack

- **Language**: Vanilla JavaScript (ES2020+), no dependencies, no build step
- **Crypto**: Web Crypto API (`crypto.subtle`) for AES-GCM, PBKDF2, RSA-OAEP
- **SHA-3**: Custom pure-JS Keccak-f[1600] implementation (SHA3-256 and SHA3-512)
- **Storage**: `localStorage` for fingerprint blacklist only
- **UI**: Pure HTML/CSS — single self-contained `.html` file
- **Fonts**: Bebas Neue (display), DM Mono (code/UI), Instrument Serif (italic body)
- **Server requirement**: None — file can be opened directly from disk

---

## Browser Compatibility

| Browser | Support | Notes |
|---------|---------|-------|
| Chrome 90+ | ✅ Full | Recommended |
| Firefox 90+ | ✅ Full | |
| Safari 15+ | ✅ Full | |
| Edge 90+ | ✅ Full | Chromium-based |
| Opera 76+ | ✅ Full | |
| IE 11 | ❌ None | No Web Crypto API |
| Chrome (file://) | ⚠️ Partial | Web Crypto requires secure context (HTTPS or localhost) |

> **Note:** For local use (`file://` protocol), some browsers restrict `crypto.subtle`. Serve via `localhost` for full functionality: `python3 -m http.server 8080`

---

## Developer Notes

### Running Locally

```bash
# Simple HTTP server (Python)
python3 -m http.server 8080

# Or with Node.js
npx serve .

# Open in browser
open http://localhost:8080/1life.html
```

### Extending with New Cipher Modes

Each mode follows a consistent pattern:

```javascript
// 1. Define magic bytes (6 bytes recommended)
const MY_MAGIC = new Uint8Array([...]);

// 2. Implement encrypt
async function myEnc(plain, pass) {
  // ... derive key, encrypt, return concat(MY_MAGIC, ...)
}

// 3. Implement decrypt
async function myDec(blob, pass) {
  if (!ctEq(blob.slice(0,6), MY_MAGIC)) throw new Error("Wrong format");
  // ... parse, decrypt, return plaintext
}

// 4. Register in detectMode()
if (ctEq(b.slice(0,6), MY_MAGIC)) return 'my-mode';

// 5. Add mode card in HTML, handle in doEncrypt()/doDecrypt() switch
```

### localStorage Schema

```json
// Key: "1life_v2_fps"
// Value: JSON array of SHA-256 hex strings
["a3f9b2c1...", "d7e4a891...", "..."]
```

Clearing used fingerprints (for testing):
```javascript
localStorage.removeItem('1life_v2_fps');
```

---

## Credits

```
╔══════════════════════════════════════╗
║   DEVELOPED BY                       ║
║   AdiscLabs                          ║
║   @Aditya Bhosale                    ║
╠══════════════════════════════════════╣
║   1LIFE v2.0                         ║
║   ODE Protocol — One-Decrypt         ║
║   Encryption Standard                ║
╚══════════════════════════════════════╝
```

**ODE Protocol** — designed and implemented by AdiscLabs

**Cryptographic primitives:**
- AES-256-GCM via Web Crypto API (W3C standard)
- SHA3-256 / SHA3-512 — custom Keccak-f[1600] implementation
- PBKDF2-SHA256 via Web Crypto API
- RSA-OAEP-2048 via Web Crypto API

---

## License

This project is released for educational and research purposes. The ODE protocol specification is open. The implementation may be freely used, modified, and distributed with attribution to **AdiscLabs / @Aditya Bhosale**.

**Use in production systems handling sensitive data is not recommended without a formal security audit** — particularly in light of the known loopholes documented above.

---

*"The only truly secure system is one that is powered off, cast in a block of concrete and sealed in a lead-lined room with armed guards."*
*— Gene Spafford*
