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
SHA-256: 0978392BB282B8522C1651FC9BFE862672BED4BBF656AB02E892DA16DACCD2C7
```

The profile was audited against the exact raw kernel and the recovered
`vmlinux.elf`. Three firmware-derived constants were corrected during that
audit: `P0_KERNEL_PHYS_LOAD` (`0x80080000`, Qualcomm ABL), the
`boot_id` ctl-table data pointer (`0x023762f0`), and the
`worker_thread` saved-return caller offset (`0x000db1a0`).

This profile is a controlled device-validation candidate. It is not in the
runtime support feed because no exact-release KernelSU late-load artifact has
been built and audited. An artifact for another model or full kernel release
must not be substituted.
