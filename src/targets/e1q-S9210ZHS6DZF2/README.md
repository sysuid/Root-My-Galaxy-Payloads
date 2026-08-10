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
SHA-256: 3F1671A287599297CD95D578B59B03752D5B93BB63E74504D958A051B0946CD4
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

Device log 20260810-191514 (first FRESH run) showed stage-2 leaking a valid
`mm_struct` (`object_index=25`) that the copied S24-family `27..30` object
window rejected (`min=27`), blocking FRESH before any gate attempt.  That
window was calibrated for `MM_STRUCT_SZ=0x400` (e1s/e2s); e1q's
image-verified `MM_STRUCT_SZ=0x3c0` yields a denser grid and on-device leaks
land at object_index 8..25, so the window is now `0..33`.  `APP_FOPS_MIN_OBJECT_INDEX`
stays at `24` (unverified FOPS path unchanged).

Device log 20260810-192328 showed stage-2 leak 0/3 all failing while every
thread scanned the full 64GB identity (`candidates=8388608`/thread, ~5s per
leak); the app's 45s p0 timeout killed the attempt (137) after only 3 retries
of the 8 configured setup attempts.  Following the validated S24-family
profiles (e1s/e2s set `APP_KERNEL_PAGE_KSNITCH_IDENTITY_END`), the stage-2
KernelSnitch search is now bounded to `0xffffff8000000000`..`0xffffff8900000000`
(36GB, covering the on-device leak range 0xffffff80..0xffffff88) with
`APP_KERNEL_PAGE_KSNITCH_EXACT_PARTITION`.  The object index handed to
`kernelsnitch_set_search_bounds` is clamped to the 4K mm slab object count
(e1q `MM_SLAB_ORDER=0`/`0x3c0` → 4 objects/slab) so its ASSERT holds, while
the post-leak in-page window stays `0..33`.

Device log 20260810-194950 showed stage-2 leaks now succeeding 4/4 inside
the bounded window, but `APP_RECLAIM_MAX_DIRECT_BASE 0xffffff8080000000`
(the S24-family copy, calibrated for e1s/e2s Exynos low-2GB mm slabs)
rejected every high-address candidate (e.g. `leaked=ffffff883a048000`
`base=ffffff883a048000` `object_index=8`, also 12/17) with `mm reclaim
candidate rejected base=ffffff883a048000 max=ffffff8080000000`, discarding
successful stage-2 leaks before any gate attempt.  e1q Snapdragon leaks land
in 0xffffff80..0xffffff88, so the ceiling is now `0xffffff8900000000` (matching
`APP_KERNEL_PAGE_KSNITCH_IDENTITY_END`).

Device log 20260810-200258 confirmed the ceiling fix (stage-2 now reaches the
gate write in attempts 7/15/16: `mm leaked=ffffff883a04a000 object_index=8`,
`ffffff8834965000 object_index=21`, `ffffff883767a000 object_index=8`; all
`p0 physical write status=0 ok=1`), but the p0 gate still `hits=0 changed=0`
and stage-1 pipe leak became the dominant failure (15/24 attempts died at
`pipe page child did not report base`).  `setup_kernelsnitch()` (stage-1)
now applies the same 36GB identity bounds + slab-object clamp as stage-2, so
pipe pages and the stage-2 payload live in the same 0xffffff80..0xffffff88
region (a stage-1 leak outside the 36GB window previously made overlap
impossible by construction) and stage-1 scans are ~2x faster.

Device log 20260810-201408 confirmed the stage-1 bound works
(`candidates=4718592`) and stage-1 succeeded in attempt 3
(`max_candidate=ffffff8843ff6000` -> `p0 pipe oracle prepared base=ffffff8843ff0000`),
but the whole payload was killed (137) during attempt 3's stage-2 before any
gate attempt and no attempt 4 ran -- the runner aborts within an external
wall-clock cap, and the hardcoded 5s quiet retry delay was pure budget waste
(24 x 5s = 120s of sleep).  The quiet delay is now 1s default (env
`QUIET_RETRY_DELAY_SEC`), and `prepare_pipe_buffer_page` logs stage-1
`elapsed_ms` so the real time split is visible.  The identity window is NOT
narrowed further: run 200258 (unbounded stage-1) found pipe leaks at
`ffffff8923ec0000`/`ffffff894b620000` just above 0xffffff8900000000, so the
36GB bound is kept to avoid cutting off stage-1 leak locations.

Device log 20260810-202406 (quiet delay now 1s -> 5 attempts ran) confirmed
stage-1 is fast (`pipe KernelSnitch page child base=ffffff8834960000 direct=1
elapsed_ms=5435`) and that the whole payload is killed (137) during attempt
5's stage-2 find_collisions with no attempt 6 -- an external cap at ~45-60s
means only the first 1-2 attempts can reach the gate, yet stage-1 succeeded
only 1/5 (attempts 1-4 died at stage-1).  `slide_leak_physical_base` now
retries stage-1 in-attempt (`APP_P0_PIPE_ORACLE_ATTEMPTS 3`, ~27s total within
the 45s p0 timeout), lifting per-attempt stage-1 success to ~66%.

Device log 20260810-202712 showed NO `p0 pipe oracle stage-1 try=N/3` lines
(even on failures) -- the run consumed a device-local payload cache, not the
pushed 1A238959 artifact which already contained the stage-1 retry.  A
success-path log (`p0 pipe oracle stage-1 try=N/3 ok base=...`) was added so
a run's log unambiguously shows which artifact is deployed.  This run also
had stage-2 leak 0/6 (all threads `max_matches=2/4`, one 3/4 at
`ffffff88a57a8780`), confirming the mm-leak reliability is a per-run variable
affecting both stages (in 200258 stage-2 leaked 3/3 when reached).  Ensure
the device fetches the fresh payload (clear the local files/payloads cache)
before retesting.

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
