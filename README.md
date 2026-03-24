# kasada-security-research

**Technical analysis of Kasada's client-side bot protection architecture**

> ⚠️ **Educational Research Only**: This repository contains security research for educational purposes. Bypassing bot protection systems may violate terms of service and applicable laws.

## Quick Summary

This research explores Kasada's architecture, challenge-response mechanisms, and cryptographic token binding. Key findings:

- CD tokens generated for one endpoint work universally across all endpoints
- Challenge solving happens entirely client-side in obfuscated p.js
- Server validates via HMAC-SHA256 cryptographic binding
- Browser context (fingerprinting) is essential for token generation
- Token headers must be non-null or validation fails

## Full Analysis

## Full Analysis

### 1. Kasada Architecture Overview

#### System Components
Kasada deploys three core components:
- **p.js**: Obfuscated client-side SDK (VM bytecode + string encryption)
- **/tl endpoint**: Challenge metadata endpoint (returns empty body)
- **Server-side validator**: HMAC verification + token tracking

#### Challenge-Response Flow
Unlike traditional server-challenge models, Kasada:
1. Sends p.js (obfuscated) to client
2. Client generates internal challenges via VM bytecode
3. Client solves PoW locally
4. Client submits proof via headers (x-kpsdk-cd, x-kpsdk-ct, x-kpsdk-h)
5. Server validates cryptographic signatures

### 2. Kasada's Proof-of-Work System

#### Challenge Data (CD) Structure
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
Where:

• answers[]: Solutions to cryptographic puzzles
• duration: Time spent solving (prevents pre-computation)
• d: Difficulty parameter
• workTime/st/rst: Timestamp triplet for server-side validation

PoW Validation

Server verifies:

1. Timestamp freshness: now() - CD.st < 5 seconds
2. Uniqueness: CD.id not in consumed token set
3. Cryptographic signature: HMAC-SHA256(secret, CT || CD) == x-kpsdk-h
4. Solution correctness: Answers match puzzle set

3. Kasada's Token Binding Strategy

Client Token (CT) - Expensive

• Cost: High (fingerprinting + crypto ops)
• Lifetime: ~30 minutes
• Contains: TLS fingerprint, User-Agent signature, device signals
• Reusable: YES (across multiple requests)

Challenge Data (CD) - Cheap

• Cost: Low (PoW computation)
• Lifetime: <5 seconds
• Contains: PoW answers + metadata
• Reusable: NO (single-use, tracked server-side)

HMAC Binding
```x-kpsdk-h = HMAC-SHA256(server_secret, CT || CD)```

This cryptographic binding prevents:
• CT forgery without secret
• CD modification after signing
• CT/CD mixing between sessions

4. p.js Obfuscation Techniques
Kasada employs multiple layers:

Layer 1: VM Bytecode
• Challenge-solving algorithm compiled to custom VM instructions
• Prevents direct code analysis
• Requires VM instruction set reverse-engineering

Layer 2: String Encryption
• All hardcoded strings encrypted
• Decrypted at runtime
• Prevents static string matching

Layer 3: Control Flow Flattening
• Linear code converted to non-linear execution
• Goto-based jumps
• Makes algorithm flow unclear

Layer 4: Dead Code Injection
• Non-executing code paths interspersed
• Misleads automated analysis tools
• Increases reverse-engineering time

5. Key Research Findings
Finding 1: Endpoint Universality
# CD tokens generated for endpoint A successfully validate on endpoint B
Implication: Server validates cryptographic binding, not endpoint-specific state
```
POST /api/v1/endpoint-a with CD → x-kpsdk-h verified ✓
POST /api/v2/endpoint-b with same CD → x-kpsdk-h still valid ✓
```

Finding 2: Empty /tl Response
/tl endpoint returns empty body (0 bytes) with challenge headers

This confirms: Challenge solving happens entirely client-side in p.js
• Server sends no puzzle parameters
• No challenge data in response
• All computation local to client

Finding 3: Browser Context Dependency
CD generation requires authenticated browser session
Fresh/isolated profiles fail Kasada checks despite valid auth tokens:
• Lacks browsing history
• Missing TLS fingerprint signals
• No device reputation data
• Kasada's CT fingerprinting detects this

Finding 4: Header Validation Strictness
Empty/null Kasada headers cause 403 rejection
Server validates header presence and consistency:
• x-kpsdk-h: HMAC signature (required)
• x-kpsdk-v: SDK version (required)
• x-kpsdk-cd: Challenge data (required)
• x-kpsdk-ct: Client token (required)
• Sending null values triggers WAF rejection

6. Cryptographic Assessment
Security Model
Kasada's model relies on three pillars:
1. Cryptographic Binding (HMAC-SHA256)
• Resistant to forgery without server secret
  • Prevents token tampering
2. Computational Cost (PoW)
  • Real-time solving required
  • Pre-computed answers fail timestamp validation
3. Temporal Constraints (5-second TTL)
  • Prevents capture-and-replay attacks
  • Forces synchronization with server time

Attack Surface Analysis
Strengths:
• HMAC prevents token forgery
• Timestamp validation prevents replay
• Unique ID tracking prevents reuse
• PoW adds computational barrier

Weaknesses:
• Client-side generation leaks algorithm to client
• No endpoint pinning (CD migrates between endpoints)
• Narrow timestamp window (exploitable in sync scenario)
• Browser fingerprinting can be spoofed with proper TLS context

7. Implications for Web Security
Kasada represents a shift in bot protection philosophy:
• Traditional: Server generates unpredictable challenges
• Kasada: Client generates deterministic challenges (with client-side obfuscation)

This model assumes:
• Obfuscation is sufficient defense against reverse-engineering
• Client-side PoW is computationally expensive for bots
• Cryptographic binding is unbreakable

However, the architecture inherently enables sophisticated attackers to:
1. Monitor the challenge-solving process
2. Identify PoW answer patterns
3. Intercept CD tokens before transmission
4. Replay tokens across endpoints

Conclusion
Kasada is a sophisticated bot protection system that shifts computational burden to clients while maintaining cryptographic validation server-side. However, the reliance on client-side obfuscation and the leakage of PoW mechanisms to the client create an asymmetry where determined attackers can observe, analyze, and intercept the challenge-solving process.

The system's security ultimately depends on the strength of its obfuscation rather than cryptographic principles, making it vulnerable to advanced reverse-engineering techniques.

## Research Disclaimer

This work is intended for:
- Security researchers
- Bot protection system designers
- Academic study of cryptographic systems
- Understanding modern authentication challenges

**Not for**:
- Bypassing security systems for malicious purposes
- Violating website terms of service
- Unauthorized access to protected resources

---

*If you found this research valuable, consider citing it or contributing improvements.*
