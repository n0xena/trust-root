# How an fTPM Fails to Serve as Hardware Binding of a Trust Root

## Scenario

This document describes an attempt to bind a trust root to specific hardware using a system's fTPM. The setup involves:

- Generating a new OpenPGP trust root using a YubiKey (as an HSM).
- Generating an X.509 PIV key on the same or another HSM.
- Using the system's fTPM to attest:  
  **"This key generation ceremony happened on this hardware."**

---

## Workflow

### 1. Setup PGP Trust Root

- Generate a new OpenPGP key on a YubiKey.
- Use YubiKey attestation (if available) to prove the key was generated on the device.

### 2. Setup X.509 PIV Key

- Generate a new PIV key for signing on the same or a different YubiKey (or another HSM).
- Use the YubiKey’s attestation mechanism to prove the key origin.
- Create a self-signed certificate with the `CA` flag set.

### 3. Cross-Sign for Mutual Trust

- Cross-sign the OpenPGP and PIV keys to establish mutual trust.
- Now both keys are:
  - Cryptographically linked.
  - Tied to known, attested hardware.

### 4. The fTPM Joins the Ceremony

1. Retrieve the Endorsement Key (EK) and its certificate.
2. Generate an Attestation Key (AK).
3. Perform the attestation handshake using:
   - `MakeCredential`
   - `ActivateCredential`
4. Generate a random primary key in the owner hierarchy.
5. Generate an unrestricted signing key under the primary key.
6. Certify the signing key using the AK.
7. Verify the attestation chain.
8. Use the PIV key from step 2 to issue a certificate that includes:
   - The EK public key.
   - The AK name.
   - The signing key name.
9. Optionally, cross-sign the PGP and PIV keys using the TPM-generated signing key.

---

## Where the fTPM Crashes the Party

Although TPM attestation works **at the time of the ceremony**, it **fails for later verification** in fTPMs because:

- **AKs are volatile** and **lost after reboot**.
- **fTPMs do not support persistent handles** or re-importing AKs.
- A new AK:
  - Has a **different name** (hash of key parameters).
  - **Cannot prove it is the same** AK used in prior attestations.

Consequences:

- 🔴 Attestations involving the lost AK **can no longer be verified**.
- 🔴 Certificates referencing the AK name **no longer prove anything**.
- 🔴 `tpm2_print` or similar tools **cannot verify the original AK's name** without the original AK.

---

## Failure Mode

Because the Attestation Key (AK):

- Cannot be re-imported,
- Is lost after fTPM reset or reboot,
- Cannot be deterministically regenerated,

... any proof of hardware-bound key generation **becomes unverifiable** after reboot.

---

## Conclusion

fTPMs are **not suitable** as long-term hardware roots of trust for cryptographic key ceremonies. Their limitations include:

- No support for persistent AKs.
- No non-volatile storage for AK handles or metadata.
- No ability to re-import or reconstruct an AK post-reboot.

In contrast, **discrete TPMs** with NV storage and persistent handle support **may be suitable** for this use case.

---

## References

- [TPM 2.0 Library Specification (TCG)](https://trustedcomputinggroup.org/resource/tpm-library-specification/)
- [tpm2-tools GitHub](https://github.com/tpm2-software/tpm2-tools)
- [Microsoft fTPM Overview](https://learn.microsoft.com/en-us/windows/security/information-protection/tpm/trusted-platform-module-overview)
- [Yubico PIV Attestation](https://developers.yubico.com/PIV/Attestation/)

---

update
instead of rely on presistent signature try to verify the systems integrity

---

## 🆕 Alternative Approach to AK Persistence via Attestation Snapshots

The original idea of binding a system's identity to a fTPM-based Attestation Key (AK) encountered a fundamental limitation: **non-persistence of the AK across reboots**. Since the AK is transient and lost upon system restart, it cannot serve as a stable trust anchor for long-term identity binding or re-verification of prior attestations.

### ⬇ New Approach: Snapshotting TPM Context During Active Session

Instead of persisting the AK itself, we snapshot its **runtime context** and generate a verifiable log of its **relationship to the system state** at a specific point in time.

This is achieved by:

1. **Loading the AK and a signing key** into the fTPM during a trusted session.
2. Using `tpm2_readpublic` to extract the **public name** of both keys.
3. Generating a **TPM attestation** (e.g., via `tpm2_quote` or `tpm2_certify`) over selected PCRs, optionally including a fresh nonce.
4. **Outputting all relevant data (public keys, key names, attestation, and PCR values) to disk or console**.
5. **Signing this data with a persistent cryptographic identity**, e.g., a PIV or OpenPGP key stored on a YubiKey.

The resulting bundle (e.g., a `.tar.gz` archive with a detached PGP signature) forms a cryptographically verifiable snapshot of the system's TPM state at the time of attestation.

### 🔁 Re-verification After Reboot

Even though the AK is destroyed after reboot, the attestation remains valid as a **signed record of past TPM state**. A verifier can:

- Confirm that the AK used in the attestation was loaded at the time of the session.
- Match the **quoted PCR values** against expected system measurements.
- Verify the cryptographic binding to the YubiKey identity that signed the snapshot.

### ✅ Trust Outcome

This method does not provide **device identity persistence**, but it **authenticates the platform state at a given time** and allows a remote party to determine:

- That the system was in a known-good configuration,
- That a specific, ephemeral AK was active,
- And that a trusted YubiKey holder vouches for that state.

Together, these assertions can be used to build a **social- or peer-rooted trust relationship**, without reliance on centralized certificate authorities or TPM manufacturer-issued keys.
