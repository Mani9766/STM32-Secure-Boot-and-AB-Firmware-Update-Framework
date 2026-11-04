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
