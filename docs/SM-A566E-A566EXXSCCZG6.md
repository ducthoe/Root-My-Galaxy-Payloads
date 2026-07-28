# SM-A566E A566EXXSCCZG6 target record

## Firmware identity

```text
model: SM-A566E
device: a56x
display build: BP4A.251205.006.A566EXXSCCZG6
fingerprint: samsung/a56xnsxx/a56x:16/BP4A.251205.006/A566EXXSCCZG6_OXECCZG6:user/release-keys
kernel release: 6.6.102-android15-8-abA566EXXSCCZG6-4k
kernel version: Linux version 6.6.102-android15-8-abA566EXXSCCZG6-4k (kleaf@build-host) (Android (11368308, +pgo, +bolt, +lto, +mlgo, based on r510928) clang version 18.0.0 (https://android.googlesource.com/toolchain/llvm-project 477610d4d0d988e69dbc3fae4fe86bff3f07f2b5), LLD 18.0.0) #1 SMP PREEMPT Thu Jul  9 11:02:17 UTC 2026
SDK: 36
ABI: arm64-v8a
page size: 4096
```

## Target derivation

The local `boot.img` was unpacked and recovered to:

```text
ida/boot-kernel.bin
ida/vmlinux.elf
ida/vmlinux.btf
ida/vmlinux.nm
```

IDA was kept at imagebase 0. Runtime kernel addresses are therefore
`KIMAGE_TEXT_BASE + kaslr_slide + *_OFF`.

Important target constants:

```text
KIMAGE_TEXT_BASE: 0xffffffc080000000
P0_PAGE_OFFSET:   0xffffff8000000000
P0_PHYS_OFFSET:   0x80000000
DIRECT_MAP_BASE:  0xffffff8000000000
VMEMMAP_START:    0xfffffffe00000000
sched_blocked_reason tracepoint id: 109
worker_thread post-schedule caller offset: 0x000d98dc
```

The P0 oracle fingerprint table maps the actual slide to bytes read at raw
offset `P0_ORACLE_PROBE_OFFSET - slide`. Without this conversion the raw sample
is misread as the slide and the fops direct-map verification reads zero.

## Exploit validation

The A56 app-domain payload was built as:

```text
artifacts/essi-A566EXXSCCZG6/cve-2026-43499-app.so
size: 104128
SHA-256: 6bfd75d1d8f87c2d1ecd159c122fed89f96c93d84d2537f89f60f631188b550d
```

Hardware execution succeeded on the connected SM-A566E. The successful run
resolved slide `0x00080000`, installed pipe physrw, and transitioned the shell
UID from 2000 to 0. The root daemon command:

```sh
/data/local/tmp/cve-2026-43499-root -c id
```

returned:

```text
uid=0(root) gid=0(root) groups=0(root) context=u:r:kernel:s0
```

## KernelSU build and crash record

KernelSU was built from tag `v3.2.5`, commit
`b0bc817b4e966aa6aa830834eaf6ef765d821d40`, with the repository Samsung
KDP/RKP/DEFEX patch.

The module was built in:

```text
ghcr.io/ylarod/ddk-min:android15-6.6-20260313
```

The DDK release was overridden to the exact target release:

```text
vermagic: 6.6.102-android15-8-abA566EXXSCCZG6-4k SMP preempt mod_unload modversions aarch64
```

The target config has:

```text
CONFIG_MODULES=y
CONFIG_MODVERSIONS=y
CONFIG_MODULE_SIG_ALL=y
# CONFIG_MODULE_SIG_FORCE is not set
# CONFIG_MODULE_FORCE_LOAD is not set
CONFIG_TRIM_UNUSED_KSYMS=y
CONFIG_KALLSYMS_ALL=y
CONFIG_KDP=y
CONFIG_RKP=y
CONFIG_SECURITY_DEFEX=y
```

The stripped standalone KO is:

```text
kernelsu/android15-6.6_kernelsu-A566EXXSCCZG6-kdp.ko
size: 327528
SHA-256: ff18d067fe8d2ecb9ea7710d68856ae4c0033a0e37329c78926246b8dbff97a0
```

Static checks passed:

```text
__versions size: 0
undefined symbols: 221
module version entries: 0
missing from target symbol table: 0
symbols resolved from kallsyms rather than target exports: 67
target CRC mismatches: 0
```

The Android/AArch64 `ksud` binary embeds the A56 KO as
`android15-6.6_kernelsu.ko`:

```text
kernelsu/ksud-A566EXXSCCZG6-kdp
size: 4765976
SHA-256: f15ee6d9900155ec8ade6d098de64a86408734211103b5826e3a6d3d1474a2fb
```

The first hardware `late-load` run was attempted through the root daemon's
DEFEX-safe bind-mount route. The phone rebooted. Recovered Samsung logs in
`/sdcard/log/recovery` show repeated bad page table entries in process `toybox`
followed by:

```text
Unable to handle kernel paging request at virtual address 0000007901000400
PC is at lookup_swap_cgroup_id+0x38/0x74
Kernel panic - not syncing: Oops: Fatal exception
```

No KernelSU control fd was verified after that run. Treat this KernelSU pair as
build/static-audit complete, not hardware-complete.
