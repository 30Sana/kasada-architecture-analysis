# 🛡️ Kasada Security Research

<p align="center">
  <b>Technical analysis of Kasada's client-side bot protection architecture</b>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Research-Security-blue" />
  <img src="https://img.shields.io/badge/Focus-Bot%20Protection-orange" />
  <img src="https://img.shields.io/badge/Status-Educational%20Only-red" />
  <img src="https://img.shields.io/badge/License-Research-lightgrey" />
</p>

---

> ⚠️ **Educational Research Only**
> This repository contains security research for educational purposes.
> Bypassing bot protection systems may violate terms of service and applicable laws.

---

## 📌 Quick Summary

This research explores **Kasada's architecture**, **challenge-response mechanisms**, and **cryptographic token binding**.

### 🔍 Key Findings

* ✅ CD tokens generated for one endpoint work across all endpoints
* 🧠 Challenge solving occurs entirely client-side in obfuscated `p.js`
* 🔐 Server validation uses **HMAC-SHA256 cryptographic binding**
* 🌐 Browser context (fingerprinting) is required for token generation
* ❗ Token headers must be non-null or validation fails

---

## 🧠 Full Analysis

---

### 1. 🏗️ Architecture Overview

#### 🔧 System Components

Kasada deploys three core components:

* **`p.js`** → Obfuscated client-side SDK (VM bytecode + string encryption)
* **`/tl` endpoint** → Challenge metadata endpoint *(returns empty body)*
* **Server-side validator** → HMAC verification + token tracking

#### 🔄 Challenge-Response Flow

Unlike traditional server-driven challenges:

1. Client receives `p.js`
2. Internal challenges generated via VM bytecode
3. Proof-of-Work solved locally
4. Proof submitted via headers:

   * `x-kpsdk-cd`
   * `x-kpsdk-ct`
   * `x-kpsdk-h`
5. Server validates cryptographic integrity

---

### 2. ⚙️ Proof-of-Work System

#### 📦 Challenge Data (CD) Structure

```json
{
  "workTime": 1774345743062,
  "id": "e15a0bdf6b2e3fcbd48d926a2f2f",
  "answers": [9, 2],
  "duration": 94.8,
  "d": -1987,
  "st": 1774339362592,
  "rst": 1774339360605
}
```

#### 🧾 Field Breakdown

* `answers[]` → Puzzle solutions
* `duration` → Solve time (anti pre-compute)
* `d` → Difficulty parameter
* `workTime / st / rst` → Timestamp validation triplet

#### ✅ Server Validation Logic

1. **Timestamp freshness**
   `now() - CD.st < 5 seconds`

2. **Uniqueness**
   `CD.id` must not be reused

3. **Signature validation**

   ```
   HMAC-SHA256(secret, CT || CD) == x-kpsdk-h
   ```

4. **Solution correctness**
   Answers must match expected puzzle output

---

### 3. 🔐 Token Binding Strategy

#### 💠 Client Token (CT) — *Expensive*

* High computational cost
* Lifetime: ~30 minutes
* Includes:

  * TLS fingerprint
  * User-Agent signature
  * Device signals
* ♻️ Reusable across requests

#### 💠 Challenge Data (CD) — *Cheap*

* Low computational cost
* Lifetime: < 5 seconds
* Contains PoW answers + metadata
* 🚫 Single-use only

#### 🔑 HMAC Binding

```
x-kpsdk-h = HMAC-SHA256(server_secret, CT || CD)
```

#### 🔒 Security Guarantees

* Prevents CT forgery
* Prevents CD tampering
* Prevents cross-session token mixing

---

### 4. 🧩 p.js Obfuscation Techniques

Kasada uses layered obfuscation:

#### 🧱 Layer 1: VM Bytecode

* Custom virtual machine instructions
* Prevents direct code inspection

#### 🔑 Layer 2: String Encryption

* Runtime decryption of all strings
* Blocks static analysis

#### 🔀 Layer 3: Control Flow Flattening

* Non-linear execution (goto-style)
* Obscures logical flow

#### 🧪 Layer 4: Dead Code Injection

* Non-executing paths
* Misleads analysis tools

---

### 5. 🔎 Key Research Findings

#### 🧭 Endpoint Universality

CD tokens are **not endpoint-bound**:

```http
POST /api/v1/endpoint-a → ✅ Valid
POST /api/v2/endpoint-b → ✅ Still valid
```

**Implication:** Validation depends on cryptographic binding, not endpoint context.

---

#### 📭 Empty `/tl` Response

* Returns **0-byte body**
* Only headers included

**Conclusion:**

* No server-side challenge data
* All computation happens client-side

---

#### 🌐 Browser Context Dependency

Token generation requires a real browser environment:

* Browsing history
* TLS fingerprint
* Device reputation

🚫 Fresh profiles fail validation even with valid auth

---

#### 🚫 Header Validation Strictness

Missing or null headers → **403 Forbidden**

Required headers:

* `x-kpsdk-h` → HMAC signature
* `x-kpsdk-v` → SDK version
* `x-kpsdk-cd` → Challenge data
* `x-kpsdk-ct` → Client token

---

### 6. 🧮 Cryptographic Assessment

#### 🛡️ Security Model

Kasada relies on:

1. **Cryptographic Binding**

   * HMAC-SHA256 integrity protection

2. **Computational Cost**

   * Real-time PoW requirement

3. **Temporal Constraints**

   * ~5 second TTL prevents replay

---

#### ⚖️ Attack Surface Analysis

**✅ Strengths**

* HMAC prevents forgery
* Timestamp limits replay
* Unique IDs prevent reuse
* PoW adds computational barrier

**⚠️ Weaknesses**

* Client-side logic exposure
* No endpoint binding
* Tight timing window
* Fingerprinting can be spoofed

---

### 7. 🌍 Implications for Web Security

#### 🔄 Paradigm Shift

| Traditional                  | Kasada                      |
| ---------------------------- | --------------------------- |
| Server-generated challenges  | Client-generated challenges |
| Opaque logic                 | Obfuscated logic            |
| Server-controlled difficulty | Client-executed PoW         |

---

#### ⚠️ Architectural Tradeoffs

This model assumes:

* Obfuscation resists reverse engineering
* PoW slows automation
* Cryptographic binding is sufficient

However, attackers can:

1. Observe challenge execution
2. Extract PoW patterns
3. Intercept CD tokens
4. Replay across endpoints

---

## 🧾 Conclusion

Kasada represents a **modern, client-heavy bot protection system** that shifts computational burden to users while maintaining server-side cryptographic validation.

However:

* Client-side execution exposes logic
* Obfuscation becomes the primary defense layer
* Advanced attackers can observe and replay behavior

👉 The system’s security depends more on **obfuscation strength** than purely on cryptographic guarantees.

---

## ⚖️ Research Disclaimer

### ✅ Intended For

* Security researchers
* Bot protection designers
* Academic study
* Authentication system analysis

### ❌ Not Intended For

* Bypassing protections maliciously
* Violating terms of service
* Unauthorized system access

---

<p align="center">
  ⭐ If you found this research useful, consider contributing or citing it.
</p>
