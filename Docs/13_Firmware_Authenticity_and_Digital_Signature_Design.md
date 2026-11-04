# Firmware Authenticity and Digital Signature Design

## 1. Introduction

The existing STM32 Secure Boot and A/B Firmware Update Framework implements
SHA-256-based firmware integrity verification.

SHA-256 is used to calculate the firmware image digest and compare it against
the reference digest stored in firmware metadata. This helps detect firmware
modification when the reference digest is trusted.

However, SHA-256 alone does not establish the origin of the firmware.

An attacker who can modify both the firmware image and its stored reference
digest may be able to bypass this integrity check.

To address this limitation, digital-signature-based firmware authentication
is introduced as a security enhancement.

The objective is to ensure that the bootloader accepts firmware only when
its authenticity can be verified using a trusted public key.

## 2. Integrity vs. Authenticity

These are two different but complementary security properties.

### 2.1 Firmware Integrity

Integrity answers the question:

**Has the firmware been modified?**

SHA-256 generates a fixed 256-bit digest from the firmware image.

The bootloader calculates the digest and compares it against the trusted
reference digest.

If both values match, the firmware has passed the integrity check.

However, the digest itself does not identify who generated it.

### 2.2 Firmware Authenticity

Authenticity answers the question:

**Was the firmware signed by an authorized source?**

Digital signatures use asymmetric cryptography consisting of two related keys:

- Private key: Used by the authorized firmware developer to sign firmware.
- Public key: Used by the bootloader to verify the signature.

The private key must remain confidential and must never be embedded in
the firmware or distributed with the bootloader.

The corresponding public key is provisioned in the bootloader and acts as
a trust anchor.

A valid signature establishes that the signed data was authorized by
the holder of the corresponding private key, assuming the private key
has not been compromised and the public key is trusted.

### 2.3 Why Both Are Required

SHA-256 and digital signatures serve complementary purposes.

| Security mechanism | Purpose |
|--------------------|---------|
| SHA-256 | Calculates the firmware digest |
| Digital signature | Establishes the authenticity of signed data |
| Public-key verification | Verifies the signature against a trusted key |

The proposed implementation retains the existing SHA-256 integrity
verification and adds digital-signature-based authenticity verification.

Neither mechanism provides firmware confidentiality or cryptographically
enforced anti-rollback protection by itself.

## 3. Firmware Authentication Approaches

There are two common approaches to cryptographic firmware authentication.

### 3.1 Symmetric-Key Authentication (HMAC)

HMAC uses a shared secret key to generate and verify a message
authentication code.

Both the host-side signing tool and the target device must possess the
same secret key.

**Advantages:**
- Relatively simple implementation.
- Lower computational requirements compared with many public-key operations.
- Small authentication tag.

**Limitations:**
- The secret key must be protected on every device that performs verification.
- Compromise of a shared device key can enable unauthorized firmware
  authentication.
- Secure provisioning and distribution of the shared key become important
  operational challenges.

### 3.2 Asymmetric-Key Authentication (Digital Signature)

Asymmetric authentication uses a private/public key pair.

The firmware developer uses the private key to generate a digital signature.
The bootloader uses the corresponding public key to verify it.

**Advantages:**
- The private signing key does not need to be stored on the target device.
- The public key can be distributed to multiple devices.
- Compromise of one device's public key does not reveal the private signing key.
- Supports centralized firmware signing and distributed verification.

**Limitations:**
- Greater computational requirements than HMAC.
- Requires cryptographic-library integration.
- Private-key protection and public-key provisioning must be addressed.

### 3.3 Selected Approach

Asymmetric digital-signature-based authentication is preferred for this
project.

The firmware signing operation will be performed on the host PC, while
signature verification will be performed by the STM32F407 bootloader.

The private key will remain outside the target device. Only the trusted
public key will be provisioned in the bootloader.

## 4. Digital Signature Algorithm Evaluation

The following algorithms were considered for firmware authenticity
verification:

- RSA (Rivest-Shamir-Adleman)
- ECDSA (Elliptic Curve Digital Signature Algorithm)
- EdDSA (Edwards-curve Digital Signature Algorithm)

The selection criteria include:

- Cryptographic security strength.
- Signature and public-key size.
- Flash and RAM requirements.
- Computational complexity on ARM Cortex-M4.
- Availability of embedded cryptographic libraries.
- Host-side signing-tool support.
- Compatibility with the existing SHA-256 implementation.

### 4.1 RSA with SHA-256

RSA is a widely established asymmetric cryptographic algorithm.

For firmware authentication, RSA-PSS with SHA-256 is a suitable
candidate.

RSA uses large integer arithmetic and modular exponentiation.

Two relevant key sizes are:

| Parameter | RSA-2048 | RSA-3072 |
|-----------|----------|----------|
| Modulus size | 2048 bits | 3072 bits |
| Signature size | 256 bytes | 384 bytes |
| Approximate security strength | 112 bits | 128 bits |

RSA is mature, widely supported and remains a valid choice for secure
firmware boot implementations.

However, its signature and public-key representations require more storage
than ECDSA P-256 for comparable security strength.

### 4.2 ECDSA P-256 with SHA-256

ECDSA is a digital-signature algorithm based on elliptic-curve cryptography.

P-256, also known as secp256r1 or prime256v1, is the elliptic curve used
for this evaluation.

Its relevant characteristics are:

| Parameter | ECDSA P-256 |
|-----------|-------------|
| Curve | NIST P-256 / secp256r1 |
| Private key | 32 bytes |
| Public key | 64 bytes (raw X and Y coordinates) |
| Raw signature | 64 bytes (R and S values) |
| Hash algorithm | SHA-256 |
| Approximate security strength | 128 bits |

The actual serialized public-key and signature sizes depend on the encoding.
For example, DER-encoded ECDSA signatures have variable length.

ECDSA offers the following advantages for the STM32F407 project:

- Compact signature representation.
- Smaller public-key storage requirements.
- Approximately 128-bit security strength.
- Compatibility with SHA-256.
- Availability of established embedded cryptographic implementations.

ECDSA signing requires correct handling of the per-signature nonce.
Deterministic ECDSA, as specified in RFC 6979, is an option to avoid
dependence on a fresh random nonce for each signing operation.

### 4.3 EdDSA (Ed25519)

Ed25519 is a digital-signature scheme based on Edwards-curve cryptography.

Its characteristics include:

| Parameter | Ed25519 |
|-----------|---------|
| Private key seed | 32 bytes |
| Public key | 32 bytes |
| Signature | 64 bytes |
| Internal hash | SHA-512 |
| Approximate security strength | 128-bit class |

Ed25519 provides compact keys and deterministic signing.

It is a viable embedded-security alternative. However, its internal
SHA-512 processing differs from the existing SHA-256-based firmware
integrity implementation.

The firmware digest can still be calculated using SHA-256 and included
in a signed manifest.

### 4.4 Algorithm Comparison

| Criteria | RSA-3072 | ECDSA P-256 | Ed25519 |
|----------|----------|------------|---------|
| Approximate security strength | 128 bits | 128 bits | 128-bit class |
| Signature size | 384 bytes | 64 bytes (raw) | 64 bytes |
| Public-key size | 384-byte modulus | 64 bytes (raw X/Y) | 32 bytes |
| Signing | Private key | Private key | Private key |
| Verification | Public key | Public key | Public key |
| Deterministic signing | Depends on scheme | Supported through RFC 6979 | Built-in |
| SHA-256 integration | Natural with RSA-PSS | Natural | External SHA-256 digest can be included in a signed manifest |
| Embedded suitability | Suitable | Suitable | Suitable candidate |

The table compares standard key and signature representations.
Actual Flash, RAM, stack and execution-time requirements depend on
the selected library and implementation.
