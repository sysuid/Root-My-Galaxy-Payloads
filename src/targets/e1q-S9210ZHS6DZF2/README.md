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
SHA-256: 9BDB60BB3381FCB52A050786B1CE62662DD83EF79ADDB42612B5C1504457CEB2
```

Build variant is `e1q-S9210ZHS6DZF2-app-physical-p0-oracle-fresh`: the P0
oracle session was switched to the S24-family FRESH mode
(`APP_REQUIRE_FRESH_P0_SESSION` + `APP_P0_REFRESH_ORACLE_EACH_FRESH_PAGE`).
The non-FRESH single-shot path never overlapped the stage-1 pipe pages with
the stage-2 skb payload page (separate freed-pool rounds -> `gate hits=0
changed=0` on every attempt reaching the p0 write). FRESH re-runs the pipe
oracle + payload page per fresh attempt, re-shaping the heap so the gate
write at `pipebuf_page_base+0x800` gets a fresh chance to land on a
pipe_buffer `.page`. Fingerprint stays generator-forward; `MM_SLAB_ORDER=0`,
futex hash `0x400` and the `4096/128/8` KernelSnitch profile remain
device-validated.

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
