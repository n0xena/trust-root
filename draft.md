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

