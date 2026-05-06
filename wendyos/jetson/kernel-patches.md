# Jetson Kernel Patches

WendyOS applies out-of-tree patches to the `linux-jammy-nvidia-tegra` kernel used on Jetson/Tegra targets. Patches live under `meta-tegra-extensions/recipes-kernel/linux/linux-jammy-nvidia-tegra/` and are applied in numbered order during the Yocto build.

## CVE-2026-31431 — AF_ALG AEAD "Copy Fail"

**Severity:** CVSS 7.8 (CISA KEV)

A logic bug in the `algif_aead` kernel subsystem allows an unprivileged local user to trigger a controlled 4-byte write into the page cache of any readable file via the AF_ALG AEAD socket interface. The root cause is the in-place AEAD optimization introduced in upstream commit `72548b093ee3` (2017).

WendyOS backports the fix from the `linux-5.15.y` stable series (2026-04-30). Five patches are applied in order:

| Patch file | Upstream commit | Description |
|---|---|---|
| `0001-crypto-scatterwalk-Backport-memcpy_sglist.patch` | `36435a56cd6b` | Backports `memcpy_sglist()` from upstream for scatter-gather list copying |
| `0002-crypto-algif_aead-use-memcpy_sglist-instead-of-null-skcipher.patch` | `17774d99bb43` | Replaces the null skcipher copy path in `algif_aead` with `memcpy_sglist()` |
| `0003-crypto-algif_aead-Revert-to-operating-out-of-place-CVE-2026-31431.patch` | `19d43105a97b` | Reverts the in-place AEAD optimization; fixes CVE-2026-31431 |
| `0004-crypto-algif_aead-snapshot-IV-for-async-AEAD-requests.patch` | `a920cabdb0b7` | Snapshots the IV into per-request storage to prevent async IV aliasing |
| `0005-crypto-algif_aead-Fix-minimum-RX-size-check-for-decryption.patch` | `fd427dd84f22` | Fixes the minimum RX buffer size check for decryption requests |

Patches are taken verbatim from `gregkh/linux linux-5.15.y`. Apply order matters — each patch depends on the previous one.

### What the fix does

`algif_aead` reverts to out-of-place operation: the AD (associated data) is copied directly from the TX scatter-gather list into the RX buffer, and the AEAD request sources from the TX SGL while writing to the RX SGL. The `aead_tfm` wrapper struct and the dependency on `CRYPTO_NULL` are removed; the module now holds a plain `struct crypto_aead *`.

### References

- [NVD CVE-2026-31431](https://nvd.nist.gov/vuln/detail/CVE-2026-31431)
- [Red Hat RHSB-2026-02](https://access.redhat.com/security/vulnerabilities/RHSB-2026-02)
