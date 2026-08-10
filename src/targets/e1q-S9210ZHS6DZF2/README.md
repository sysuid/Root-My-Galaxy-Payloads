# e1q-S9210ZHS6DZF2 device-validation status

```text
device: Samsung Galaxy S24 (SM-S9210, e1q)
firmware: S9210ZHS6DZF2
fingerprint: samsung/e1qzhx/e1q:16/BP4A.251205.006/S9210ZHS6DZF2:user/release-keys
kernel release: 6.1.145-android14-11-33419968-abS9210ZHS6DZF2
```

`target.h` and `p0_fingerprint.h` are derived from the exact raw kernel
documented in `docs/SM-S9210-S9210ZHS6DZF2.md`. The P0 table has 32 candidate
slide rows and covers 256 source qwords.

The checked-in app artifact is:

```text
artifacts/e1q-S9210ZHS6DZF2/cve-2026-43499-app.so
size: 104128
SHA-256: C4EFDD59CC1D283F1DF0AF8C6D238CE2C443366C239FA9F633F8B9E20B06EB14
```

The profile was audited against the exact raw kernel and the recovered
`vmlinux.elf`. Three firmware-derived constants were corrected during that
audit: `P0_KERNEL_PHYS_LOAD` (`0x80080000`, Qualcomm ABL), the
`boot_id` ctl-table data pointer (`0x023762f0`), and the
`worker_thread` saved-return caller offset (`0x000db1a0`).

This profile is published in the runtime support feed (`support/targets-v3.json`)
with the exact-release KernelSU late-load pair. The P0 fingerprint row ordering
was re-audited byte-for-byte against the raw Image and restored to the
generator-forward convention (`row S = X[(0x1f0000 − S)]`), correcting an
earlier e3q-mirrored reversal that contradicted the Image. An artifact for
another model or full kernel release must not be substituted.
