# Security — HSM, PCI DSS, Encryption in Financial Systems

> "In financial security, the question is never 'can we afford the security?' The question is 'can we afford the breach?' One breach at a payment processor can cost $100M+ in fines, remediation, and reputation damage." — PCI QSA (Qualified Security Assessor)

---

## At a Glance

| | |
|---|---|
| **PCI DSS** | Payment Card Industry Data Security Standard — mandatory for card processing |
| **HSM** | Hardware Security Module — tamper-proof device that manages encryption keys |
| **Tokenization** | Replace card number with a random token — store token, not card data |
| **Encryption** | AES-256 at rest, TLS 1.3 in transit — all cardholder data encrypted |

---

## The Fundamental Principle: Never Store What You Don't Need

```
Senior to Junior:
"The safest cardholder data is the cardholder data you never store.
 Every byte of card data you store is a liability:
  - PCI compliance scope expands
  - Breach impact grows
  - Regulatory exposure increases

 Modern approach: tokenize immediately at the edge.
 Only the payment processor (Stripe, Omise) stores card data.
 Your app stores only the TOKEN (useless to attackers)."
```

---

## HSM — Hardware Security Module

### What It Is

```
An HSM is a dedicated cryptographic device that:
  - Stores encryption keys in tamper-proof hardware (keys NEVER leave the device)
  - Performs cryptographic operations (encrypt, decrypt, sign, verify)
  - Self-destructs if physically tampered (keys erased on tampering detection)
  - Processes operations in microseconds (dedicated cryptographic ASICs)

Without HSM:
  Encryption keys stored in application memory or configuration files
  If server is compromised: attacker gets the key + the encrypted data = full breach

With HSM:
  Encryption keys stored inside tamper-proof hardware
  If server is compromised: attacker gets encrypted data but NO key = useless
  HSM only decrypts when the application authenticates to it
  HSM physically destroys keys if tampered with
```

### HSM in the Payment Flow

```
Card Processing Flow WITH HSM:

1. Customer types card number on checkout page
   → Encrypted in browser using Stripe.js (never hits your server)
   → OR: card number transmitted to payment terminal

2. Payment terminal uses HSM to:
   → Validate PIN (HSM performs PIN verification, PIN never leaves HSM)
   → Derive session key (HSM generates unique key per transaction)
   → Encrypt card data (HSM encrypts, key never leaves HSM)

3. Encrypted card data travels to your payment processor
   → Even if intercepted: encrypted with key only in HSM

4. Payment processor's HSM:
   → Decrypts card data (using their HSM key)
   → Performs authorization
   → Issues CVV/PIN verification
   → All within the HSM — raw card data never in server memory

HSM vendors used by banks:
  Thales Luna (HSBC, Deutsche Bank, BNP Paribas)
  Futurex Vectera Plus (US banks, Visa/Mastercard processors)
  AWS CloudHSM (cloud-native, used by fintech startups)
  Azure Dedicated HSM (Microsoft cloud)
  nCipher nShield (many European banks)
```

---

## PCI DSS — The Law of Card Processing

PCI DSS (Payment Card Industry Data Security Standard) is maintained by the PCI Security Standards Council (founded by Visa, Mastercard, AmEx, Discover, JCB).

### The 12 Requirements

```
Requirement 1:  Install and maintain firewall/network security controls
Requirement 2:  Do not use vendor-supplied defaults for system passwords
Requirement 3:  Protect stored cardholder data (encrypt or don't store)
Requirement 4:  Protect cardholder data with strong cryptography in transit
Requirement 5:  Protect all systems and networks from malicious software
Requirement 6:  Develop and maintain secure systems and software
Requirement 7:  Restrict access to system components by business need
Requirement 8:  Identify users and authenticate access to system components
Requirement 9:  Restrict physical access to cardholder data
Requirement 10: Log and monitor all access to network resources and cardholder data
Requirement 11: Test security of systems and networks regularly
Requirement 12: Support information security with organizational policies
```

### PCI Compliance Levels

```
Level 1: > 6 million card transactions/year
  → Annual on-site audit by Qualified Security Assessor (QSA)
  → Quarterly network scan by Approved Scanning Vendor (ASV)
  → Who: Visa, Mastercard, Stripe, PayPal, large banks

Level 2: 1–6 million transactions/year
  → Annual Self-Assessment Questionnaire (SAQ) + QSA review
  → Quarterly ASV scan
  → Who: mid-size merchants, regional payment processors

Level 3: 20,000–1 million e-commerce transactions/year
  → Annual SAQ
  → Quarterly ASV scan

Level 4: < 20,000 e-commerce OR < 1 million all channels
  → Annual SAQ
  → Recommended quarterly ASV scan
  → Most small businesses
```

### Cardholder Data — What You Can and Cannot Store

```
Card data:
  PAN (Primary Account Number) = the 16-digit card number
  Cardholder Name
  Expiration Date
  Service Code

Sensitive Authentication Data (SAD) — NEVER store after authorization:
  Full magnetic stripe data
  CVV/CVC/CVV2/CID (the 3-4 digit security code)
  PIN / PIN block

What you CAN store (with encryption + PCI controls):
  PAN (masked to first 6 + last 4: 4111-XXXX-XXXX-1111)
  Cardholder name
  Expiration date

What most apps SHOULD do:
  Store NOTHING — use tokenization instead
  Stripe, Omise, 2C2P: give you a token
  Store the token (useless to attackers, useful to you for recurring charges)
```

---

## Encryption Architecture for Financial Systems

### Encryption at Rest

```
Tier 1: Database encryption (AES-256)
  PostgreSQL: pgcrypto or Transparent Data Encryption (TDE)
  RDS: encryption enabled at instance level (AWS KMS key)
  All card data, PII, account numbers encrypted at rest

Tier 2: Column-level encryption
  Specific sensitive columns encrypted with application-managed keys
  Even DBA cannot read raw values — only encrypted ciphertext
  Key stored in HSM or AWS KMS (not on the same server as data)

Tier 3: File-level encryption
  Database backups encrypted before storing to S3
  Encryption key separate from the backup

Best practice (Stripe, Monzo approach):
  Envelope encryption:
    Data encrypted with Data Encryption Key (DEK)
    DEK encrypted with Key Encryption Key (KEK) stored in HSM/KMS
    Only the encrypted DEK is stored with the data
    To decrypt: KMS decrypts the DEK, then DEK decrypts data
    Even if database is stolen: data is protected (no DEK)
    Even if DEK is stolen: KEK is in HSM (not accessible)
```

### Encryption in Transit

```
All financial APIs: TLS 1.3 minimum (never TLS 1.0 or 1.1)
  TLS 1.3 improvements over 1.2:
    - Forward secrecy by default (past traffic can't be decrypted if key compromised)
    - Faster handshake (1-RTT vs 2-RTT)
    - Removed weak ciphers (RC4, 3DES, MD5)

Internal service-to-service: mTLS (mutual TLS)
  Both client AND server present certificates
  Server authenticates client → no impersonation attacks
  Used by: Istio service mesh, all service-to-service in payment systems

Certificate pinning (mobile apps):
  App hard-codes the expected server certificate fingerprint
  Even if attacker has a valid CA certificate → won't match pinned cert
  Man-in-the-middle becomes impossible
  Used by: banking apps worldwide

API security:
  Signed requests: HMAC-SHA256 signature on request body
  Prevents: request tampering in transit
  Used by: Stripe webhooks, AWS API, most financial APIs
```

---

## Tokenization — The Modern Standard

```
How tokenization works:

Customer enters card: 4111-1111-1111-1111 (real PAN)
                              │
                              ▼
                    [Tokenization Service] (e.g., Stripe, Omise)
                      - Stores real PAN in their vault (PCI Level 1)
                      - Returns token: tok_abc123xyz (random, useless without vault)
                              │
                              ▼
               Your application stores: tok_abc123xyz
               Your database: NEVER sees the real card number
               Your PCI scope: MASSIVELY reduced (you don't handle card data)

When you need to charge again:
  Send tok_abc123xyz to Stripe → Stripe looks up real PAN → charges
  Your system never touches the real card number

Network tokenization (next level):
  Visa Token Service / Mastercard Digital Enablement Service
  Replaces PAN with network token (specific to merchant)
  Even if token is stolen: only works for that one merchant
  Revocable: token can be invalidated without changing the physical card
  Used by: Apple Pay, Google Pay, Samsung Pay
```

---

## In Clean Architecture Terms

```
Security = Cross-cutting concern (every layer has security responsibilities)

Presentation Layer (Controller):
  - Validate input (reject malicious input before it reaches use case)
  - Apply rate limiting (prevent brute force)
  - Authenticate requests (JWT, API key, session)
  - Never log sensitive data (no PANs in application logs)

Use Case Layer:
  - Never store raw card data (call ITokenizationService instead)
  - Enforce business-level authorization ("can this user access this account?")
  - Generate idempotency keys

Interface Adapters:
  ITokenizationService → StripeTokenizationAdapter
    (Stripe handles PCI scope for you)
  IEncryptionService → HSMEncryptionAdapter
    (HSM handles key management for you)

Infrastructure Layer:
  TLS termination at load balancer
  mTLS between all internal services
  Database encryption via KMS
  HSM-backed key storage

Clean Architecture benefit:
  ITokenizationService (port) defined in use case layer
  If you switch from Stripe to Omise:
    Change StripeTokenizationAdapter → OmiseTokenizationAdapter
    Use case code unchanged
    Business rule "tokenize before storing" unchanged
```

---

## Sources
- [HSMs for PCI DSS Compliance — PCI DSS Guide](https://pcidssguide.com/hsms-for-pci-dss-compliance/)
- [PCI Security Standards Council — HSM Standard](https://www.pcisecuritystandards.org/about_us/press_releases/pci-security-standards-council-updates-hardware-security-module-standard/)
- [What is Hardware Security Module — Fortinet](https://www.fortinet.com/resources/cyberglossary/hardware-security-module)
