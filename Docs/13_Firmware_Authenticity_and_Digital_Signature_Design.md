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

## 5. Why ECDSA P-256 Is Preferred for This Project

RSA was initially considered as a possible firmware-authentication
mechanism because of its maturity and broad industry adoption.

However, ECDSA P-256 provides a more compact signature and public-key
representation while offering approximately 128-bit security strength.

For an STM32F407-based bootloader, reducing cryptographic storage
requirements is beneficial because the bootloader has limited Flash and
RAM resources.

The selection is also supported by existing embedded secure-boot
implementations.

### 5.1 Industry and Embedded Ecosystem References

**STMicroelectronics X-CUBE-SBSFU**

STMicroelectronics provides an STM32 Secure Boot and Secure Firmware
Update reference implementation.

Its documentation describes ECDSA using the NIST P-256 curve together
with SHA-256 for firmware authentication.

This is particularly relevant because the reference solution includes
STM32F4-family support.

Reference:
[STMicroelectronics UM2262 – X-CUBE-SBSFU User Manual](https://www.st.com/resource/en/user_manual/um2262-getting-started-with-the-xcubesbsfu-stm32cube-expansion-package-stmicroelectronics.pdf)

**MCUboot**

MCUboot is an established open-source secure bootloader for
microcontrollers.

It supports several firmware-image signature algorithms, including
ECDSA P-256, RSA-2048, RSA-3072 and Ed25519.

Its image-signing utility also supports host-side key generation
and firmware signing.

References:
[MCUboot – Signed Images](https://github.com/mcu-tools/mcuboot/blob/main/docs/signed_images.md)

[MCUboot – Image Tool](https://github.com/mcu-tools/mcuboot/blob/main/docs/imgtool.md)

### 5.2 Security-Strength Comparison

According to NIST SP 800-57 Part 1 Revision 5, the approximate
128-bit security-strength category corresponds to:

- RSA: 3072-bit key.
- ECC: 256-bit curve parameter.

Therefore, ECDSA P-256 and RSA-3072 are the appropriate comparison
for approximately equivalent classical security strength.

RSA-2048 provides approximately 112-bit security strength and should
not be directly compared with P-256 as an equal-strength alternative.

Reference:
[NIST SP 800-57 Part 1 Revision 5](https://doi.org/10.6028/NIST.SP.800-57pt1r5)

### 5.3 Decision

ECDSA P-256 with SHA-256 is selected as the primary candidate for
the STM32F407 firmware-authentication implementation.

The selection is based on:

- Approximately 128-bit security strength.
- Compact public-key and signature representation.
- Compatibility with the existing SHA-256 firmware-integrity mechanism.
- Availability in established STM32 secure-boot reference implementations.
- Support in embedded cryptographic ecosystems.

RSA-PSS with SHA-256 remains a technically valid alternative.

The decision does not imply that ECDSA is always faster than RSA.
Actual verification time, Flash consumption, RAM usage and stack
requirements must be measured on the target hardware before final
cryptographic-library selection.

Ed25519 remains an alternative for further evaluation if library
compatibility and resource requirements justify it.

**Final implementation selection remains subject to cryptographic-library
evaluation and STM32F407 hardware benchmarking.**

## 6. Proposed Firmware Signing and Verification Workflow

The proposed architecture separates firmware signing from firmware
verification.

Firmware signing is performed on the authorized host PC using a private
key. Firmware authentication is performed by the STM32F407 bootloader
using the corresponding trusted public key.

The private key is never stored on the STM32 device.

### 6.1 Signing and Verification Roles

| Component | Responsibility |
|-----------|----------------|
| Host PC | Calculate firmware SHA-256 and generate digital signature |
| Private key | Sign the firmware manifest |
| Firmware package | Carry firmware image, manifest and signature |
| STM32 bootloader | Verify signature and firmware integrity |
| Public key | Verify the authenticity of the signed manifest |

### 6.2 Signed Firmware Manifest

A firmware manifest contains the security-relevant information associated
with the firmware image.

The proposed manifest fields include:

| Field | Description |
|-------|-------------|
| Manifest format version | Identifies the manifest structure version |
| Signature algorithm | Identifies the supported digital-signature algorithm |
| Hash algorithm | Identifies the firmware digest algorithm |
| Key identifier | Identifies the trusted public key used for verification |
| Firmware version | Identifies the firmware release version |
| Image size | Specifies the firmware binary size |
| Target device identifier | Identifies the compatible target device or board |
| Target slot / execution address | Identifies the intended firmware execution location |
| Firmware SHA-256 | Contains the calculated firmware image digest |

The exact manifest serialization format, field widths, byte order and
signature representation will be finalized during implementation.

The manifest must use a deterministic, well-defined byte representation.
Compiler-dependent C structure padding must not be used as the signing
format.

### 6.3 Host-Side Firmware Signing Process

The host-side signing utility performs the following operations:

1. Read the compiled firmware binary.
2. Calculate the SHA-256 digest of the firmware image.
3. Construct the firmware manifest using the image digest and associated
   firmware information.
4. Serialize the manifest using the defined byte format.
5. Generate a digital signature over the serialized manifest using the
   authorized private key.
6. Generate the signed firmware package containing the firmware image,
   manifest and signature.

For the proposed ECDSA P-256 implementation, SHA-256 will also be used
by the ECDSA signing operation to hash the serialized manifest.

The firmware image digest and the manifest digest serve different purposes:

- Firmware digest: Identifies the contents of the firmware image.
- Manifest digest: Represents the manifest data covered by the
  digital signature.

The private key remains on the authorized host-side signing environment
and must not be included in the firmware package.

### 6.4 Firmware Package Structure

The proposed logical package consists of three components:

```text
Signed Firmware Package
|
+-- Firmware Manifest
|     |
|     +-- Manifest format version
|     +-- Signature algorithm
|     +-- Hash algorithm
|     +-- Key identifier
|     +-- Firmware version
|     +-- Image size
|     +-- Target device identifier
|     +-- Target slot / execution address
|     +-- Firmware SHA-256 digest
|
+-- Digital Signature
|     |
|     +-- Signature generated using private key
|
+-- Firmware Binary
      |
      +-- Executable firmware image
```
### 6.5 STM32 Bootloader Authentication Process
After receiving a staged firmware package, the STM32 bootloader performs
the following operations:
1. Read the package header and manifest.
2. Perform initial structural, length and memory-boundary checks.
3. Verify that the signature algorithm and manifest format are supported.
4. Identify the applicable trusted public key.
5. Verify the digital signature over the serialized manifest.
6. Calculate SHA-256 over the staged firmware image.
7. Compare the calculated firmware digest against the digest contained
   in the authenticated manifest.
8. Validate target-device compatibility, image size and firmware
   version policy.
9. Proceed with inactive-slot programming only if all required checks
   succeed.
The bootloader must not trust manifest fields before successful signature
verification, except for limited parsing needed to safely locate and
validate the manifest.

Unsupported algorithms, invalid signatures, malformed packages and
invalid image boundaries must result in rejection.
### 6.6 Integration with Existing A/B Firmware Update Flow
The digital-signature verification mechanism will be integrated into
the existing STM32 A/B update architecture.

The proposed update sequence is:
```text
Running Application
        |
        v
Update Notification
        |
        v
Download / Staging Area
        |
        v
Reset into Bootloader
        |
        v
Read Firmware Package
        |
        v
Validate Package Structure and Boundaries
        |
        v
Verify Digital Signature
        |
        +------ Invalid ------> Reject Firmware
        |
        v
Calculate Firmware SHA-256
        |
        +------ Mismatch -----> Reject Firmware
        |
        v
Validate Firmware Version and Target
        |
        v
Select Inactive Firmware Slot
        |
        v
Erase and Program Inactive Slot
        |
        v
Verify Programmed Image SHA-256
        |
        +------ Mismatch -----> Reject Candidate
        |
        v
Persist Authenticated Manifest and Signature
        |
        v
Append BOOT_PENDING Metadata Record
        |
        v
Trial Boot
        |
        +------ Confirmation ------> CONFIRMED
        |
        +------ Watchdog Reset ----> Rollback
```
The inactive slot must not be erased or programmed until the staged
firmware has passed signature verification, digest verification and
the required compatibility checks.

The existing destination-image SHA-256 verification, BOOT_PENDING
state, application confirmation and rollback mechanisms will be retained.
### 6.7 Authentication During Normal Boot
Firmware authenticity must not be checked only during the update process.

The authenticated manifest and signature associated with each installed
firmware image must remain available to the bootloader.

During normal boot, the bootloader should:
1. Read the selected slot's authenticated manifest and signature.
2. Verify the signature using the trusted public key.
3. Validate the firmware image boundaries and size.
4. Calculate SHA-256 over the selected firmware image.
5. Compare the calculated digest against the digest in the authenticated
   manifest.
6. Transfer execution only after successful verification and the
   existing boot-state checks.
This ensures that an image modified after installation cannot become
trusted merely because it was previously accepted.

The storage location and lifecycle of per-slot signed manifests must be
defined without compromising the existing metadata journal.

The current 60-byte metadata journal records should be preserved where
practical. The signed manifest and signature may be maintained as
separate per-slot authentication data.
### 6.8 Authentication Failure Handling
The following conditions must result in firmware rejection:
- Missing firmware signature.
- Invalid digital signature.
- Firmware signed with an unauthorized private key.
- Modified firmware manifest.
- Firmware SHA-256 mismatch.
- Unsupported signature or manifest format.
- Invalid image size or memory boundaries.
- Incompatible target-device information.
An unauthenticated candidate must never be promoted to BOOT_PENDING
or executed.

If candidate authentication fails, the previously confirmed firmware
should remain the boot target, provided it passes the existing boot
validation checks.

If the selected installed image fails authenticity or integrity
verification, the bootloader must not execute it. Recovery behavior
must follow the defined fail-safe and fallback policy.
### 6.9 Security Properties and Limitations
The proposed digital-signature mechanism provides:
- Firmware authenticity verification.
- Integrity protection for the signed manifest.
- Cryptographic binding between firmware identification fields and
  the firmware digest.
- Verification using a trusted public key without storing the private
  signing key on the target MCU.
The mechanism does not independently provide:
- Firmware confidentiality.
- Cryptographically enforced anti-rollback protection.
- Protection against private-key compromise.
- Protection against physical replacement or modification of the
  bootloader trust anchor.
- Cryptographic authentication of the existing mutable metadata
  journal unless separately addressed.
These properties require additional security controls and are outside
the immediate scope of firmware-signature implementation.

---

## Two important architecture decisions to remember

**1. Verify the signature on every normal boot.**

This is important. Imagine an attacker modifies the installed firmware after a successful update. If your bootloader checks only the mutable metadata SHA-256 digest, the attacker might replace both the firmware and digest. Keeping the signed manifest with each slot allows the bootloader to authenticate the image again at startup.

**2. Your 60-byte metadata record needs special consideration.**

Your existing metadata record contains the firmware digest and boot state information. We should not casually increase its size or change the journal implementation. The signed manifest and signature storage should be designed alongside the existing per-slot metadata.

One more point specific to your project: because Slot A and Slot B are linked at different execution addresses, the signed manifest must bind the image to its intended slot or execution address unless we later change to a slot-independent image design.

## 7. Trusted Public-Key Management and Protection

Digital-signature-based firmware authentication depends on the secure
management of cryptographic keys.

The proposed architecture uses an asymmetric key pair consisting of
a private signing key and a corresponding public verification key.

The private key is used by the authorized firmware signing environment.
The public key is provisioned in the STM32F407 bootloader and is used
to verify firmware signatures.

The public key does not need to remain confidential. However, its
authenticity and integrity must be protected because replacing the
trusted public key could allow unauthorized firmware to be accepted.

### 7.1 Private-Key Management

The private key is the most sensitive asset in the firmware signing
architecture.

Anyone possessing the authorized private key may be able to generate
firmware signatures that the bootloader accepts.

The following practices are proposed:

- Generate the private key using an established cryptographic tool or
  library.
- Keep the private key outside the STM32 target device.
- Never include the private key in the application firmware,
  bootloader, firmware package or Git repository.
- Restrict access to the private key to the authorized signing
  environment.
- Store the private key in an encrypted and access-controlled location.
- Maintain protected backups to avoid permanent loss of signing
  capability.
- Separate development/test keys from production signing keys.

For a production environment, a hardware security module (HSM) or
equivalent protected key-management solution should be considered
to prevent direct exposure of the private key.

For the current development implementation, a locally generated
development key will be used. It must not be treated as a production
signing key.

### 7.2 Public-Key Management

The corresponding public key is used by the STM32F407 bootloader
during firmware signature verification.

Unlike the private key, the public key is not confidential.

However, it must be protected against unauthorized replacement or
modification.

The proposed approach is:

- Generate the public key from the authorized private key.
- Provision the public key as part of the trusted bootloader image.
- Store the public key in the protected bootloader region.
- Make the public key available to the signature-verification module.
- Prevent normal application and firmware-update operations from
  modifying the trusted key storage.
- Document the key identifier, algorithm and key encoding.

For the proposed ECDSA P-256 implementation, the public key is
represented by the elliptic-curve X and Y coordinates.

The exact public-key encoding will be selected according to the
cryptographic library requirements.

### 7.3 Public-Key Storage in STM32F407

The current STM32F407 memory architecture reserves:

| Flash region | Address | Purpose |
|--------------|---------|---------|
| Sectors 0-1 | 0x08000000 - 0x08007FFF | Bootloader |
| Sector 2 | 0x08008000 | Slot A metadata |
| Sector 3 | 0x0800C000 | Slot B metadata |
| Sector 4 | 0x08010000 | Download / staging |
| Sector 5 | 0x08020000 | Slot A firmware |
| Sector 6 | 0x08040000 | Slot B firmware |

The trusted public key will initially be included in the bootloader
image stored in sectors 0-1.

This approach keeps the trust anchor separate from the application
firmware, staging area and per-slot metadata.

The public key must not be stored in a region that is erased during
normal A/B firmware updates.

### 7.4 Flash Write Protection

The STM32F407 provides Flash Write Protection (WRP) using configuration
option bytes.

Write protection can be enabled for selected Flash sectors to prevent
their programming or erasure.

The proposed configuration is to protect bootloader sectors 0-1,
which also contain the trusted public key.

This protection helps prevent accidental or unauthorized modification
of the bootloader and its embedded trust anchor through normal Flash
programming operations.

The write-protection configuration must be validated on the actual
STM32F407 hardware.

### 7.5 Readout Protection (RDP)

The STM32F407 provides Readout Protection (RDP) with three protection
levels:

- Level 0: No readout protection.
- Level 1: Readout protection enabled.
- Level 2: Maximum readout protection.

RDP Level 1 restricts external debug access and access through
boot-from-RAM and system-memory boot configurations.

When booting from user Flash, user code can still access Flash memory.
Therefore, RDP Level 1 alone does not prevent application code from
modifying Flash contents.

Flash Write Protection and Readout Protection serve different purposes
and should be evaluated together.

RDP Level 2 is irreversible on the STM32F407 and disables interfaces
including JTAG and SWV. It can permanently restrict debugging and
device servicing.

Therefore, RDP Level 2 will not be used during development and testing.
Any production protection configuration requires a separate hardware
security and recovery assessment.

References:

- STM32F407 Reference Manual RM0090
- STMicroelectronics AN5156: Introduction to Security for STM32 MCUs

### 7.6 Public-Key Identification

The signed firmware manifest will contain a key identifier.

The key identifier allows the bootloader to identify which trusted
public key should be used for signature verification.

The bootloader must maintain an authorized public-key list or an
equivalent trusted-key lookup mechanism.

The key identifier itself is not proof of authenticity. The selected
public key must successfully verify the digital signature over the
complete signed manifest.

For the initial implementation, a single trusted public key is
sufficient.

Support for multiple trusted keys can be introduced as a future
extension.

### 7.7 Key Rotation and Revocation

Cryptographic signing keys may need to be replaced due to:

- Planned key lifecycle changes.
- Private-key compromise.
- Changes in firmware signing infrastructure.
- Introduction of new cryptographic keys or algorithms.

The current implementation will initially use one provisioned public
key.

Future key rotation should be designed so that a new public key is
accepted only after authorization by an existing trusted root key or
through another explicitly trusted provisioning procedure.

Simply receiving a new public key in a firmware package must never
cause the bootloader to trust it automatically.

If the original private key is compromised, an alternative trusted
recovery mechanism is required. This may involve a separately
provisioned recovery key or controlled physical re-provisioning.

Key revocation and multi-key lifecycle management are not part of the
initial implementation scope.

### 7.8 Key Management Security Requirements

The following requirements are established for the proposed design:

1. The private signing key must never be stored on the STM32F407.
2. The private key must not be committed to the source repository.
3. The public verification key must be provisioned as part of the
   trusted bootloader.
4. Normal firmware updates must not modify the trust-anchor storage.
5. Bootloader Flash sectors must be protected against unintended
   programming and erase operations.
6. The bootloader must verify firmware signatures using an authorized
   public key.
7. An unknown key identifier must result in authentication failure.
8. New public keys must not be trusted without an authorized key-update
   or provisioning procedure.
9. Development keys and production signing keys must remain separate.

### 7.9 Current Implementation Scope

The initial implementation will use:

- One ECDSA P-256 private signing key on the host PC.
- One corresponding trusted public key in the STM32F407 bootloader.
- Host-side firmware signing.
- Bootloader-side digital-signature verification.
- Flash Write Protection evaluation for bootloader sectors 0-1.

Public-key rotation, revocation, HSM integration and production
key-management infrastructure remain future enhancements.

## 8. Cryptographic Library Evaluation

After identifying ECDSA P-256 with SHA-256 as the primary
digital-signature candidate, the next step is to evaluate suitable
cryptographic libraries for implementation on STM32F407.

The library must provide reliable ECDSA signature verification,
support the selected elliptic curve and integrate with the existing
STM32CubeIDE bootloader project.

### 8.1 Evaluation Requirements

The following requirements will be considered during library selection:

1. Support for ECDSA using NIST P-256 (secp256r1).
2. Support for SHA-256 or compatibility with an external SHA-256
   implementation.
3. Signature verification functionality.
4. Compatibility with ARM Cortex-M4.
5. Compatibility with the STM32CubeIDE GCC toolchain.
6. Reasonable Flash and RAM requirements.
7. Controlled stack usage during cryptographic operations.
8. Minimal dependency on dynamic memory allocation.
9. Well-defined public-key and signature data formats.
10. Availability of documentation, examples and test vectors.
11. Maintainability and suitability for embedded firmware projects.

The evaluation will initially focus on signature verification because
the private-key signing operation will be performed on the host PC.

### 8.2 Candidate Libraries

The following libraries have been identified for evaluation.

#### A. STMicroelectronics X-CUBE-CRYPTOLIB

X-CUBE-CRYPTOLIB is the STM32 cryptographic library provided by
STMicroelectronics.

It provides cryptographic functionality for STM32 microcontrollers
and includes implementations for Cortex-M based devices.

The supported algorithms include:

- SHA-256 and other SHA-2 variants.
- ECDSA with NIST P-256 and other supported curves.
- EdDSA, including Ed25519.
- RSA signature generation and verification.
- Other cryptographic primitives.

The library provides examples for GCC-based development environments,
including STM32CubeIDE.

**Advantages:**

- Developed specifically for the STM32 ecosystem.
- Explicit support for STM32F4 series devices.
- Support for ECDSA P-256 and SHA-256.
- Provides a dedicated cryptographic interface for STM32 Cortex-M
  processors.
- Supports other algorithms if future requirements change.

**Considerations:**

- Understand the CMOX API and integration requirements.
- Identify the required library components and dependencies.
- Evaluate the generated Flash and RAM footprint.
- Measure stack consumption and signature-verification execution time.
- Verify compatibility with the existing STM32CubeIDE project.

Official references:

[X-CUBE-CRYPTOLIB – STMicroelectronics](https://www.st.com/en/embedded-software/x-cube-cryptolib.html)

[STM32 Cryptographic Library Overview](https://wiki.st.com/stm32mcu/wiki/Security:Cryptographic_library_Overview)

[ST X-CUBE-CRYPTOLIB Data Brief](https://www.st.com/resource/en/data_brief/x-cube-cryptolib.pdf)

#### B. Mbed TLS

Mbed TLS is an established cryptographic library designed for
embedded systems and constrained environments.

It provides cryptographic primitives and protocol implementations,
including elliptic-curve operations and digital-signature
functionality.

Relevant capabilities include:

- ECDSA signature generation and verification.
- NIST P-256 elliptic-curve support.
- SHA-256.
- Configurable cryptographic components.

**Advantages:**

- Broad embedded-system adoption.
- Well-documented APIs.
- Supports ECDSA P-256.
- Provides configurable features to reduce binary and memory footprint.
- Supports multiple cryptographic algorithms.

**Considerations:**

- Configuration must be optimized for a bootloader-only use case.
- Unused TLS, X.509 and other components should be excluded where
  possible.
- Evaluate the selected maintained release and its API compatibility.
- Measure Flash, RAM and stack consumption on STM32F407.
- Review dynamic memory requirements and error handling.

Official references:

[Mbed TLS Documentation](https://mbed-tls.readthedocs.io/)

[Mbed TLS – Reducing Memory and Storage Footprint](https://mbed-tls.readthedocs.io/en/latest/kb/how-to/reduce-polarssl-memory-and-storage-footprint/)

[Mbed TLS GitHub Repository](https://github.com/Mbed-TLS/mbedtls)

#### C. micro-ecc

micro-ecc is a lightweight C implementation of elliptic-curve
cryptography.

It supports ECDSA and ECDH and is designed for embedded
microcontrollers.

Relevant characteristics include:

- Support for secp256r1 (NIST P-256).
- Written in C.
- Optional ARM and Thumb assembly optimizations.
- No dynamic memory allocation.
- Small-footprint implementation.

**Advantages:**

- Lightweight compared with a general-purpose cryptographic library.
- Written in C and suitable for embedded integration.
- Supports the required P-256 curve.
- No dynamic memory allocation.
- Provides a focused ECDSA implementation.

**Considerations:**

- Does not provide a complete SHA-256 implementation.
- The existing project SHA-256 implementation must be retained
  or integrated separately.
- Signature and public-key encoding must be handled consistently.
- Security review, test-vector validation and maintenance history
  must be evaluated before adoption.
- Integration and regression testing will be required.

Official reference:

[micro-ecc GitHub Repository](https://github.com/kmackay/micro-ecc)

### 8.3 Library Comparison

| Evaluation criteria | X-CUBE-CRYPTOLIB | Mbed TLS | micro-ecc |
|---------------------|------------------|----------|-----------|
| ECDSA P-256 | Yes | Yes | Yes |
| SHA-256 | Yes | Yes | No |
| STM32F4 support | Yes | Requires integration | Requires integration |
| STM32CubeIDE GCC | Examples available | GCC-compatible | GCC-compatible |
| Dynamic memory | To be evaluated | Configurable | No dynamic allocation |
| Library scope | General cryptographic library | General cryptographic library | Focused ECC library |
| Integration complexity | To be evaluated | To be evaluated | To be evaluated |
| Resource consumption | To be measured | To be measured | To be measured |

The comparison identifies supported capabilities rather than
assuming that one library has better performance than another.

Actual resource consumption and execution time must be measured
using the same STM32F407 hardware and compiler configuration.

### 8.4 Proposed Evaluation Strategy

The libraries will be evaluated in the following order:

1. X-CUBE-CRYPTOLIB
2. Mbed TLS
3. micro-ecc

X-CUBE-CRYPTOLIB will be evaluated first because of its direct
STM32F4 support and integration with the STM32CubeIDE ecosystem.

Mbed TLS will be considered as an alternative if integration,
resource usage or API requirements make it more suitable.

micro-ecc will be considered if a smaller and more focused
ECDSA implementation is preferred.

### 8.5 Performance and Memory Benchmarking

The selected library must be evaluated on the actual STM32F407
hardware.

The following parameters will be measured:

| Parameter | Measurement objective |
|-----------|------------------------|
| Flash consumption | Additional code and constant-data size |
| Static RAM | Global and static memory requirements |
| Stack consumption | Peak stack usage during signature verification |
| Verification time | Execution time for one P-256 signature verification |
| CPU utilization | Processing cycles required for verification |
| Dependencies | Required modules, runtime support and configuration |
| Integration | Compatibility with the existing bootloader build |

For a fair comparison, the libraries should use equivalent
compiler optimization settings and the same signature test vectors.

Verification time can be measured using the Cortex-M4 DWT cycle
counter or a hardware timer.

Flash and static RAM consumption will be obtained from the
linker map file.

Peak stack usage will be measured using an appropriate stack
monitoring technique.

### 8.6 Functional Validation Requirements

Before integrating a library into the bootloader, its verification
functionality must pass standalone tests.

The test cases will include:

1. Verification of a valid ECDSA P-256 signature.
2. Rejection of an invalid signature.
3. Rejection when a different public key is supplied.
4. Rejection when signed data is modified.
5. Rejection of malformed signature input.
6. Rejection of invalid public-key data.
7. Validation using known cryptographic test vectors.

The host-side signing implementation and STM32 verification
implementation must use identical data serialization and signature
encoding conventions.

### 8.7 Preliminary Library Preference

X-CUBE-CRYPTOLIB is the preferred first candidate for evaluation
because ST provides explicit STM32F4 support and examples for
STM32CubeIDE.

However, final selection will depend on:

- Successful integration into the existing bootloader.
- Functional verification results.
- Flash and RAM consumption.
- Peak stack usage.
- Signature-verification execution time.
- API and data-format compatibility.

No library is considered finally selected until these checks
have been completed and the results documented.

### 8.8 Expected Outcome

The outcome of this evaluation will be a documented selection
of the cryptographic library used for ECDSA P-256 signature
verification on STM32F407.

The selected implementation must provide a reliable verification
mechanism without compromising the existing bootloader,
memory layout, SHA-256 integrity verification or A/B update
functionality.

### Final Signed Firmware Storage Decisions

1. The staging package consists of an 80-byte manifest, a 64-byte
   ECDSA signature and the firmware binary.

2. The staging layout is:
   Manifest → Signature → Firmware Image.

3. The installed Slot A and Slot B layout is:
   Firmware Image → Reserved Space → Manifest → Signature.

4. The manifest and signature occupy the final 144 bytes of each
   firmware slot at fixed addresses.

5. The existing 60-byte metadata journal remains unchanged initially.

6. The bootloader authenticates staged firmware before installation.

7. The bootloader authenticates the selected installed firmware
   on every reset before transferring execution.

8. When no update package is available, candidate authentication
   is skipped. Normal boot authentication is still mandatory.

9. The existing BOOT_PENDING, CONFIRMED and rollback mechanisms
   are retained.

10. The final layout must be validated against the linker scripts,
    Flash programming routines and existing staging metadata.
