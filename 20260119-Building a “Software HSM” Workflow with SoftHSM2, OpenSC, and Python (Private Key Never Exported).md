---
title: "Building a “Software HSM” Workflow with SoftHSM2, OpenSC, and Python (Private Key Never Exported)"
description: "This article explains a practical, end-to-end workflow for using SoftHSM2 as a software-backed HSM..."
pubDatetime: 2026-01-19T11:05:47.467Z
---

This article explains a practical, end-to-end workflow for using **SoftHSM2** as a software-backed HSM (PKCS#11 provider), managing it with **OpenSC** tools, and performing **encryption/decryption from Python without exporting the private key**.



## 1) What You Are Building (Roles and Responsibilities)

A clean mental model helps avoid common mistakes:

* **SoftHSM2**: A software implementation of an HSM. It stores keys inside a token and exposes cryptographic operations via **PKCS#11**. Properly configured, the **private key remains non-exportable** (you do not dump it as PEM).
* **OpenSC**: Operational tooling—most importantly `pkcs11-tool`—used to **inspect slots, initialize tokens, generate key pairs, and list objects**.
* **Python**: Your application layer, which loads a PKCS#11 module and invokes cryptographic operations such as **decrypt** (or **sign**) using the private key **inside the token**.



## 2) Core Concept: “Private Key Non-Export” and the Correct Crypto Pattern

A frequent misconception is “encrypt with private key and decrypt with public key.” That is not confidentiality encryption—conceptually it is closer to **signing**.

For confidentiality, the standard pattern is:

1. **Encrypt with the public key** (anyone can do this).
2. **Decrypt with the private key** (only the key owner can do this).

In a PKCS#11 workflow, step (2) is performed by calling **`C_Decrypt`** on the token’s private key object. The private key itself never leaves SoftHSM2.

For real-world payloads, you typically use **hybrid encryption**:

* Encrypt the actual data with **AES-GCM** (fast, handles large payloads).
* Encrypt (wrap) the AES key with the **RSA public key**.
* Decrypt (unwrap) the AES key using the **RSA private key inside the token** via PKCS#11.
* Decrypt the data with the recovered AES key.



## 3) Prerequisites: Install Tools and Locate the PKCS#11 Module

### Install (examples)

* Debian/Ubuntu:

  * `sudo apt-get install softhsm2 opensc`
* macOS (Homebrew):

  * `brew install softhsm opensc`

### Locate SoftHSM2’s PKCS#11 module

You must know where the `libsofthsm2.so` (Linux) / `.dylib` (macOS) file is.

Examples:

* `find /usr -name "libsofthsm2.so" 2>/dev/null`
* `ldconfig -p | grep softhsm` (if available)

In the examples below, we refer to the module path as:

* `PKCS11_LIB=/path/to/libsofthsm2.so`



## 4) Step 1 — SoftHSM2: Token Setup (Key Storage and Cryptographic Operations)

SoftHSM2 stores tokens as files under a configured token directory (`tokendir`). The configuration file is commonly:

* `/etc/softhsm2.conf` or
* `~/.config/softhsm2/softhsm2.conf`

You can override with `SOFTHSM2_CONF` if needed.

### 4.1 Inspect available slots

```bash
softhsm2-util --show-slots
```

### 4.2 Initialize a token (set SO PIN and User PIN)

```bash
softhsm2-util --init-token --slot 0 --label "demo-token"
```

You will be prompted for:

* **SO PIN** (administrator PIN: token management operations)
* **User PIN** (application PIN: day-to-day crypto operations)



## 5) Step 2 — OpenSC: Manage and Validate with `pkcs11-tool`

OpenSC’s `pkcs11-tool` is the most practical way to generate keys and verify objects.

### 5.1 List tokens

```bash
pkcs11-tool --module "$PKCS11_LIB" -L
```

### 5.2 Generate an RSA key pair (private key stays inside the token)

```bash
pkcs11-tool --module "$PKCS11_LIB" \
  --login --pin <USER_PIN> \
  --keypairgen --key-type rsa:2048 \
  --id 01 --label "rsa-key-01"
```

Notes:

* `--id 01` is an object identifier (treat it as bytes; manage uniqueness carefully).
* `--label` is human-friendly and often used for searches in applications.
* The private key is created as a token object and should be kept **non-exportable** by policy.

### 5.3 List objects (confirm key pair exists)

```bash
pkcs11-tool --module "$PKCS11_LIB" -O --login --pin <USER_PIN>
```

You should see entries such as:

* `Private Key Object`
* `Public Key Object`

### 5.4 Export the public key for testing/distribution (safe to export)

Export public key (DER):

```bash
pkcs11-tool --module "$PKCS11_LIB" \
  --read-object --type pubkey --id 01 \
  -o pubkey.der
```

Convert to PEM with OpenSSL:

```bash
openssl pkey -pubin -inform DER -in pubkey.der -out pubkey.pem
```



## 6) Step 3 — Python: Encrypt/Decrypt with a Non-Exported Private Key

### 6.1 Choose a Python PKCS#11 binding

Common options:

* `python-pkcs11` (more modern, higher level)
* `PyKCS11` (thin wrapper, more manual work)

Regardless of library, the application flow is the same:

1. Load the PKCS#11 module (`libsofthsm2.so`)
2. Select token by label (`demo-token`)
3. Open a session
4. Login with User PIN
5. Find the private key object by label or ID
6. Call decrypt/sign using PKCS#11 mechanisms
7. Logout and close session



## 7) Minimal Proof: Public-Key Encrypt → Token-Private-Key Decrypt

This is the simplest validation that you are actually using the token’s private key without exporting it.

### 7.1 Encrypt with OpenSSL using the exported public key

Example using RSA-OAEP with SHA-256 (plaintext must be small enough for RSA):

```bash
echo -n "hello" > plain.bin

openssl pkeyutl -encrypt -pubin -inkey pubkey.pem \
  -pkeyopt rsa_padding_mode:oaep \
  -pkeyopt rsa_oaep_md:sha256 \
  -pkeyopt rsa_mgf1_md:sha256 \
  -in plain.bin -out cipher.bin
```

### 7.2 Decrypt in Python via PKCS#11 (private key stays in token)

Your Python code will typically:

* Read `cipher.bin`
* Call PKCS#11 decrypt using the private key handle

Pseudocode outline:

```python
PKCS11_LIB = "/path/to/libsofthsm2.so"
TOKEN_LABEL = "demo-token"
USER_PIN = "1234"
KEY_LABEL = "rsa-key-01"

cipher = open("cipher.bin", "rb").read()

# 1) Load PKCS#11 module
# 2) Locate token by label
# 3) Open session, login(USER_PIN)
# 4) Find private key object by label or ID
# 5) Decrypt with RSA-OAEP-SHA256 mechanism
# 6) Output plaintext
```

Key requirement:

* The RSA padding and hash parameters must match what you used during encryption (OAEP vs PKCS#1 v1.5, SHA-256 vs SHA-1, etc.).



## 8) Recommended Production Pattern: Hybrid Encryption (RSA + AES-GCM)

For any non-trivial data size, use hybrid encryption:

1. Generate a random AES key.
2. Encrypt data with **AES-GCM** (Python).
3. Encrypt (wrap) AES key with **RSA public key** (OpenSSL or Python).
4. Decrypt (unwrap) AES key with **RSA private key inside SoftHSM2** via PKCS#11.
5. Decrypt data with AES key (Python).

This makes HSM usage efficient and standard-compliant: the HSM performs a small private-key operation; symmetric crypto handles bulk data.



## 9) Operational Notes and Troubleshooting

### SoftHSM2 security posture

SoftHSM2 is a **software HSM**. It can satisfy a “private key not exported” workflow at the API/object level, but it does not provide hardware tamper resistance. File permissions and host security are therefore critical.

### Useful verification commands

If something fails, isolate where:

* SoftHSM2 view:

  * `softhsm2-util --show-slots`
* PKCS#11 token visibility:

  * `pkcs11-tool --module "$PKCS11_LIB" -L`
* Login/object visibility:

  * `pkcs11-tool --module "$PKCS11_LIB" -O --login --pin ...`

Common issues include:

* Wrong PKCS#11 module path
* Token directory permissions (`tokendir`)
* Mismatched mechanism parameters (e.g., OAEP hash settings)
* Wrong object search criteria (label vs ID)



## Summary

By combining:

* **SoftHSM2** (PKCS#11 key storage and crypto operations),
* **OpenSC** (`pkcs11-tool` for management and validation),
* **Python** (PKCS#11 session + decrypt/sign calls),

you can implement a reliable pipeline where the **private key is never exported**, yet your application can still perform necessary cryptographic operations securely via PKCS#11.
