# Porting Droidspaces to the moto g57 power (mumba / SM6475 "parrot")

Working notes for building a Droidspaces-capable kernel for this exact device.
Everything here was established by inspecting the live phone over adb and by
reading the Droidspaces 7.1.0 tree in this repo.

## 1. What the device actually is

| Item | Value |
|---|---|
| Model | moto g57 power, codename `mumba` |
| SoC | SM6475 "parrot" (Snapdragon 6s Gen 4) |
| Android | 16, build `W1WAAS36.48-12-16-3` |
| Kernel | `6.6.87-android15-8-gf7bcb1c8e88f-ab14322536-4k` |
| Flavour | AOSP GKI 6.6, generation `android15-8`, 4 KiB pages |
| Root | KernelSU (SELinux context `u:r:ksu:s0`), Zygisk Next 1.3.4 |
| Bootloader | unlocked at runtime (`androidboot.verifiedbootstate=orange`), `securestate: flashing_unlocked` |
| Slot | `_a` |

The launcher is already installed as a Magisk/KernelSU module (v7.1.0) and the
daemon runs, so the only missing piece is the kernel.

### Partition layout that matters

| Partition | Contents | Action |
|---|---|---|
| `boot_a` | kernel only, `ramdisk_size = 0`, header v4, 1584-byte header | **replace kernel payload** |
| `init_boot_a` | 2801111-byte magisk/KSU ramdisk | leave alone, root lives here |
| `vendor_boot_a` | vendor ramdisk (7451205 B) + dtb + bootconfig | leave alone |

Because the boot ramdisk is empty, a kernel swap cannot break root: repack
`boot_a` only and `init_boot_a` keeps KSU.

## 2. Why the stock kernel cannot run Droidspaces

`droidspaces check` on the device reports **2 must-have failures**:

```
[✗] PID namespace      [✗] IPC namespace
```

Confirmed against `/proc/config.gz` — Droidspaces-relevant symbols:

| Symbol | Stock | Needed |
|---|---|---|
| `CONFIG_PID_NS` | **not set** | y (must have) |
| `CONFIG_IPC_NS` | **absent** (`CONFIG_SYSVIPC` off) | y (must have) |
| `CONFIG_SYSVIPC` | not set | y |
| `CONFIG_POSIX_MQUEUE` | not set | y |
| `CONFIG_USER_NS` | not set | y (sandboxing) |
| `CONFIG_DEVTMPFS` | not set | y (hardware mode) |
| `CONFIG_MACVLAN` | not set | y (`--net=macvlan`) |
| `CONFIG_CFS_BANDWIDTH` | not set | y (`--cpus`) |
| `CONFIG_CGROUP_PIDS` | not set | y (`--pids-limit`) |
| `CONFIG_CGROUP_DEVICE` | not set | y |
| `CONFIG_TMPFS_XATTR` / `CONFIG_TMPFS_POSIX_ACL` | not set | y (NixOS) |
| `CONFIG_NF_TABLES` | not set | y |
| `CONFIG_IP6_NF_NAT` | not set | y (NAT66) |
| `CONFIG_NAMESPACES`, `UTS_NS`, `NET_NS`, `SECCOMP*`, `CGROUPS`, `MEMCG`, `OVERLAY_FS`, `VETH`, `BRIDGE`, `NF_NAT`, `IP_NF_*`, `FUSE_FS`, `TUN`, `BLK_DEV_LOOP`, `EXT4_FS` | already set | — |

So this is a "trimmed GKI" vendor kernel: Motorola ships the ACK `gki_defconfig`
with the container-oriented blocks turned off.

## 3. Where the source comes from

Motorola publishes the kernel used here:

- Repo: <https://github.com/MotorolaMobilityLLC/kernel-msm>
- Branch matching the shipping KMI: **`android-16-release-w1va36.62-22-16`**
  (`Makefile`: 6.6.87, merge commit "GKI 'android15-6.6-2025-05_r19' for SPL 2601",
  tag `MMI-W1VA36.62-22-16`)
- Newer branch `android-16-release-w1waas36.48-12-40-4` is 6.6.102 — do **not**
  use it for this phone, it changes the KMI baseline.

Defconfig recipe (no AOSP build tree needed, plain `make` works):

```
gki_defconfig
+ arch/arm64/configs/vendor/parrot_perf.config
+ arch/arm64/configs/vendor/ext_config/moto-parrot-mumba.config
+ arch/arm64/configs/vendor/ext_config/moto-parrot.config
+ arch/arm64/configs/vendor/ext_config/moto-parrot-gki.config
+ arch/arm64/configs/vendor/ext_config/droidspaces.config   <- ours
```

Reproducing just those five files yields a `.config` that agrees with the
device's `/proc/config.gz` on every Droidspaces symbol, so the recipe is right.

Toolchain: the build log on the phone says `Android (11368308, +pgo, +bolt,
+ lto, +mlgo, based on r510928) clang`. Use exactly that prebuilt:

```
https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/tags/android-15.0.0_r1/clang-r510928.tar.gz
```

## 4. The hard constraint: vendor module CRCs

`/vendor/lib/modules/*.ko` are prebuilt, MODVERSIONS-signed objects with a
populated `__versions` section (36 entries in `adsp_loader_dlkm.ko`, e.g.
`module_layout = 0x4e276f37`). Two consequences:

1. `same_magic()` in `kernel/module/version.c` skips the release string when
   `__versions` exists, so the `-g<hash>`/date part of the version is free.
2. `check_version()` compares **symbol type CRCs**, so any kernel-side change to
   a struct layout that a vendor module imports breaks loading, loudly:
   `disagrees about version of symbol module_layout`.

That is exactly what the Droidspaces GKI patches under
`Documentation/resources/kernel-patches/GKI/` are for: they relocate
`sysv_sem`/`sysv_shm` and `user_struct.mq_bytes` into the Android kABI padding
so enabling `CONFIG_SYSVIPC` / `CONFIG_POSIX_MQUEUE` costs no ABI shift.
Applied here as `include/linux/sched.h` + `include/linux/sched/user.h` edits.

`module_layout` is the canary symbol: if a build reproduces `0x4e276f37`,
the whole `struct module`, `task_struct`, etc. baseline matches the phone.

### The trap: `CONFIG_DEBUG_INFO_BTF` changes `module_layout`

Building from `gki_defconfig` + vendor fragments gives `module_layout = 0x6500052f`
and every vendor module refuses to load. Two builds (stock config, and the
phone's own `/proc/config.gz`) both produced `0x6500052f`, so it is not the
config fragments and not the Droidspaces patches — it is one missing symbol:

| | Stock phone | Host build (no pahole) |
|---|---|---|
| `CONFIG_DEBUG_INFO_BTF` | `y` (pahole 125) | silently dropped |
| `module_layout` CRC | `0x4e276f37` | `0x6500052f` |

`struct module` carries `btf_data`/`btf_data_size` under `CONFIG_DEBUG_INFO_BTF`,
so the CRC of `module_layout` (whose signature is that struct) changes. Kconfig
drops the symbol when `pahole` is missing or older than 1.16, and pahole is easy
to miss on a stock distro — it is in `dwarves`.

**Therefore the base config must be the device's own `/proc/config.gz`, built
with pahole 1.25**, not `gki_defconfig` + fragments:

```bash
cp /proc/config.gz dump; zcat dump > device-config.txt
cp device-config.txt $OUT/.config
make -s O=$OUT ARCH=arm64 LLVM=1 CC=clang LD=ld.lld LLVM_IAS=1 olddefconfig
./scripts/kconfig/merge_config.sh -m -O $OUT $OUT/.config \
    arch/arm64/configs/vendor/ext_config/droidspaces.config
make -s O=$OUT ARCH=arm64 LLVM=1 CC=clang LD=ld.lld LLVM_IAS=1 olddefconfig
```

Get pahole without root: unpack the distro `.deb` (`pahole_1.25-0ubuntu3_amd64.deb`)
and put its `usr/bin` on `PATH`. Then verify before spending 40 minutes:

```bash
make -s O=$OUT ARCH=arm64 LLVM=1 CC=clang LD=ld.lld LLVM_IAS=1 kernel/module/version.o
grep -o "#SYMVER module_layout 0x[0-9a-f]*" $OUT/kernel/module/.version.o.cmd
# must print 0x4e276f37
```

### Every extra config option costs CRC compatibility

Starting from the phone's own config and adding the full "recommended"
Droidspaces feature list broke 103 of 484 sampled vendor symbol CRCs. Each was
traceable to one option moving a struct that leaked into a symbol signature:

| Option | What moved | Symbol that re-hashed |
|---|---|---|
| `CONFIG_CGROUP_DEVICE`, `CONFIG_CGROUP_PIDS` | `CGROUP_SUBSYS_COUNT` 7 to 9, which sizes arrays inside `struct cgroup` / `css_set`, reachable from `struct task_struct` | `__put_task_struct`, `mem_cgroup_from_id` |
| `CONFIG_BRIDGE_NETFILTER` | `struct netns_nf` gains `hooks_bridge`; `struct net` is reachable from `struct net_device` then `struct device` | `_dev_err`, every netdev helper |
| `CONFIG_NF_TABLES` | `struct net` gains `netns_nftables` | same |

So the fragment carries only the MUST HAVE switches (`PID_NS`, `SYSVIPC`,
`IPC_NS`). `droidspaces check` lists the rest as RECOMMENDED or OPTIONAL; give
them up rather than break the vendor modules. Measured result: 0 mismatches over
484 symbols taken from 8 real modules.

Probe cheaply before a full build, all three must match the phone exactly:

```bash
make -s O=$OUT ARCH=arm64 LLVM=1 CC=clang LD=ld.lld LLVM_IAS=1 \
     kernel/module/version.o drivers/base/core.o kernel/fork.o
grep -o '#SYMVER module_layout 0x[0-9a-f]*'      $OUT/kernel/module/.version.o.cmd  # 0x4e276f37
grep -o '#SYMVER _dev_err 0x[0-9a-f]*'           $OUT/drivers/base/.core.o.cmd      # 0x3b00ee3e
grep -o '#SYMVER __put_task_struct 0x[0-9a-f]*'  $OUT/kernel/.fork.o.cmd            # 0xc6040a84
```

### Two options that must be turned off

`CONFIG_MODULE_SIG_PROTECT` is on in the stock config because Motorola's build
signs its modules with a key that does not exist outside their build. With it
on, `kernel/module/main.c` refuses symbol lookups from any module whose
signature does not verify, which here is all 442 of them, for about 129 imported
vmlinux symbols. `CONFIG_TRIM_UNUSED_KSYMS` then drops every export missing from
the `abi_symbollist.raw` KMI list: `submit_bio`, `proc_create`,
`mem_cgroup_from_id` and dozens more. Both are disabled in the fragment. Cost is
a slightly larger `Image` and a taint on module load; benefit is a phone that
still has working hardware.

## 5. Build

```bash
export PATH=<pahole>/usr/bin:<clang-r510928>/bin:$PATH
cd moto-kernel                      # branch dsp-6.6.87
cp device-config.txt $OUT/.config                       # phone's /proc/config.gz
make -s O=$OUT ARCH=arm64 LLVM=1 CC=clang LD=ld.lld LLVM_IAS=1 olddefconfig
./scripts/kconfig/merge_config.sh -m -O $OUT $OUT/.config \
    arch/arm64/configs/vendor/ext_config/droidspaces.config
make -s O=$OUT ARCH=arm64 LLVM=1 CC=clang LD=ld.lld LLVM_IAS=1 olddefconfig
make -j4 O=$OUT ARCH=arm64 LLVM=1 CC=clang LD=ld.lld LLVM_IAS=1 Image
```

Keep `ARCH=arm64` on every `make` call: drop it and Kconfig quietly answers the
x86 questions, then the build fails a long way from the cause.

`~40 min` on 4 cores, `Image` ≈ 31 MB.

## 6. Package and flash

```bash
adb shell su -c 'dd if=/dev/block/by-name/boot_a of=/data/local/tmp/boot_a.img'
adb pull /data/local/tmp/boot_a.img
python3 boot_repack.py boot_a.img $OUT/arch/arm64/boot/Image boot_new.img
adb reboot bootloader
fastboot flash boot boot_new.img       # or: fastboot flash boot_a
fastboot reboot
```

`boot_repack.py` copies the original header verbatim and only swaps
`kernel_size` + payload; a same-kernel round trip is byte-identical to the
dumped partition, which is the safety check worth keeping.

Keep `init_boot_a` untouched so KernelSU survives. `fastboot flash boot`
writes the active slot; the image is 36 MB inside a 96 MB partition.

## 7. Verifying

```bash
adb shell su -c '/data/local/Droidspaces/bin/droidspaces check'
adb shell su -c 'grep -c . /proc/modules'     # expect ~442, the stock count
adb shell su -c 'dmesg | grep -i "disagrees about version"'
```

`droidspaces check` must show every MUST HAVE item green. Expected yellows,
because those options were dropped to protect the vendor ABI: `devtmpfs`,
`macvlan`, `IPv6 NAT`, `CPU limit` (`CFS_BANDWIDTH`), `Process limit`
(`CGROUP_PIDS`), `Sandboxing` (`USER_NS`). `--net=nat` still probes Bridge and
Veth green.

Then prove the runtime end to end:

```bash
adb push alpine.tar.gz /data/local/tmp/
adb shell su -c 'mkdir -p /data/local/Droidspaces/Rootfs/alpine && \
  cd /data/local/Droidspaces/Rootfs/alpine && tar xzf /data/local/tmp/alpine.tar.gz'
adb shell su -c '/data/local/Droidspaces/bin/droidspaces \
  -r /data/local/Droidspaces/Rootfs/alpine -n alpine-test --net=none start'
adb shell su -c '/data/local/Droidspaces/bin/droidspaces --name=alpine-test \
  run -- sh -c "cat /etc/alpine-release; hostname; uname -r"'
```

Measured on the device after flashing: Alpine 3.20.3, hostname `alpine`, PID 1
is the container's own `init` while the host sees it as an ordinary PID,
`6.6.87-4k-g6c5cd71a3007-dirty`.

## 8. Fallback

`kernel-images/boot_a.img` is the raw stock partition dump (the whole 96 MB), so
recovery is one command and needs no repacking:

```bash
fastboot flash boot_a kernel-images/boot_a.img
```

`kernel-images/k_orig.bin` is the same kernel extracted, for reference. Keep
`init_boot_a` untouched so KernelSU survives.

Bootloader is unlocked; if a bad kernel bootloops, hold Vol- + Power to reach
fastboot, or `fastboot boot` a good image without writing.
