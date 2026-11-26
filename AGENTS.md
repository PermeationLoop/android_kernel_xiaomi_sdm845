# AGENTS.md — LineageOS 22.2 sdm845 + KernelSU (BakaSU) + SUSFS 4.9 port

This branch carries the KernelSU + SUSFS integration for **Xiaomi Mi MIX 2S
(polaris, sdm845, kernel 4.9.337)** on top of `LineageOS/android_kernel_xiaomi_sdm845`
branch `lineage-22.2`.

Everything below was validated on a real device (LineageOS 22.2 NIGHTLY-polaris,
Android 15): KernelSU root works, SUSFS v2.3.0 initialises, ReZygisk + Zygisk
modules load, microG works (the ROM supports signature spoofing).

## Pinned state

| Component | Pin |
|---|---|
| Kernel base | `LineageOS/android_kernel_xiaomi_sdm845` @ `aa8adfe9bf21` (lineage-22.2) |
| KernelSU | git submodule `KernelSU` → `Baka-SU/BakaSU` (formerly ReSukiSU) tag `v4.2.0-rc3` (`239e1e88`) |
| Manager APK | `ReSukiSU_v4.2.0-rc3_35171-arm64-v8a-release.apk` (versionCode 35171) |
| SUSFS | `wxx9248/susfs4ksu` branch `kernel-4.9-v2`, kernel patch **v2.3.0** |
| Hook mode | SUSFS inline hook (`CONFIG_KSU_SUSFS=y`) |
| Defconfig | `arch/arm64/configs/mix2s_defconfig` (stock config + KSU/SUSFS options) |

KernelSU version code = `30000 + <git rev-list --count HEAD> + 700` of the
`KernelSU` submodule. At `v4.2.0-rc3` that is `30000 + 4471 + 700 = 35171`.
The manager APK's versionCode must be >= the kernel's; otherwise the manager
shows a version mismatch. **The submodule must have full history** — a shallow
submodule makes the computed version code wrong.

## Re-integrate on upstream updates

### 1. Kernel (LineageOS)

```sh
git remote add upstream https://github.com/LineageOS/android_kernel_xiaomi_sdm845
git fetch upstream lineage-22.2
git merge upstream/lineage-22.2        # or rebase
```

Then re-check the SUSFS patch still applies (see step 3) and that the KSU wiring
survived:

- `drivers/kernelsu` → symlink to `../KernelSU/kernel`
- `drivers/Makefile`: `obj-$(CONFIG_KSU) += kernelsu/`
- `drivers/Kconfig`: `source "drivers/kernelsu/Kconfig"`
- `fs/susfs.c`, `include/linux/susfs.h`, `include/linux/susfs_def.h` present

### 2. KernelSU / BakaSU

```sh
git -C KernelSU fetch --tags --unshallow    # keep full history
git -C KernelSU checkout <new-tag>
```

Pick the tag that has a matching manager APK release, and update the manager APK
on the device. Check that the new tag still builds against the SUSFS port: the
KSU kernel code calls `CMD_SUSFS_*`/`susfs_*` symbols provided by the SUSFS
patch. If upstream KSU adds a SUSFS command the 4.9 port does not implement
(e.g. `KSU_SUSFS_PRIORITY_UID_FILTER` / `susfs_set_sus_path_uid_filter`), either
backport that SUSFS feature or disable the corresponding `KSU_SUSFS_*` config.

### 3. SUSFS 4.9 port

```sh
git -C <susfs4ksu> fetch && git -C <susfs4ksu> checkout kernel-4.9-v2
patch -p1 --forward --dry-run < <susfs4ksu>/kernel_patches/50_add_susfs_in_kernel-4.9.patch
patch -p1 --forward             < <susfs4ksu>/kernel_patches/50_add_susfs_in_kernel-4.9.patch
cp <susfs4ksu>/kernel_patches/fs/susfs.c fs/
cp <susfs4ksu>/kernel_patches/include/linux/susfs.h \
   <susfs4ksu>/kernel_patches/include/linux/susfs_def.h include/linux/
```

Known post-patch fix: the patch inserts
`nd->state |= ND_STATE_OPEN_LAST;` in `do_last()` (`fs/namei.c`) before the
function's declarations. The kernel builds with
`-Werror=declaration-after-statement`, so move that block below `int error;`.
Re-check after every SUSFS update.

SUSFS `Kconfig` options live in `KernelSU/kernel/Kconfig`; the build fails if
`CONFIG_KSU_SUSFS` is selected while the kernel is not SUSFS-patched.

### 4. Defconfig changes

After changing any Kconfig option:

```sh
make O=out ARCH=arm64 CC=clang-19 mix2s_defconfig   # sanity
scripts/config --file out/.config -e KSU -e KSU_SUSFS ...
make O=out ARCH=arm64 olddefconfig
make O=out ARCH=arm64 CC=clang-19 savedefconfig && cp out/defconfig arch/arm64/configs/mix2s_defconfig
```

## Build from a clean checkout

Build on a case-sensitive filesystem. The tree contains case-colliding files
(`net/netfilter/xt_TCPMSS.c` vs `xt_tcpmss.c`, several uapi headers); building
on a case-insensitive mount (e.g. a macOS virtiofs workspace) silently corrupts
the checkout.

Host packages (Debian/Ubuntu): `clang-19 lld-19 llvm-19 bc bison flex
libssl-dev libelf-dev rsync cpio lz4 zstd python3` (add
`gcc-arm-linux-gnueabihf` only if you re-enable `CONFIG_COMPAT_VDSO`).

```sh
git clone -b lineage-22.2 https://github.com/PermeationLoop/android_kernel_xiaomi_sdm845.git
cd android_kernel_xiaomi_sdm845
git submodule update --init --recursive
# full submodule history: KSU's version code is 30000 + commit count + 700
git -C KernelSU fetch --unshallow --tags 2>/dev/null || true

# must print 35171 for v4.2.0-rc3; anything else means shallow submodule history
echo $((30000 + $(git -C KernelSU rev-list --count HEAD) + 700))

make O=out ARCH=arm64 CC=clang-19 mix2s_defconfig

make -j"$(nproc)" O=out ARCH=arm64 \
  CC=clang-19 LD=ld.lld-19 AR=llvm-ar-19 NM=llvm-nm-19 \
  OBJCOPY=llvm-objcopy-19 OBJDUMP=llvm-objdump-19 STRIP=llvm-strip-19 \
  CROSS_COMPILE=aarch64-linux-gnu- CLANG_TRIPLE=aarch64-linux-gnu- \
  HOSTCC=gcc HOSTCXX=g++ \
  Image.gz-dtb
```

Output: `out/arch/arm64/boot/Image.gz-dtb` (Image.gz + appended DTBs).
The kernel release is `<version>-perf-g<sha>` with no `-dirty` suffix as long as
the tree and submodule are committed and clean. Delete `out/` when switching
configs so a stale `.config` is not reused.

`CROSS_COMPILE_ARM32`/`arm-linux-gnueabihf-gcc` are **not** needed with the
shipped defconfig (`CONFIG_COMPAT_VDSO=n`); the check in `arch/arm64/Makefile`
only runs when `CONFIG_COMPAT_VDSO=y`.

### Check: expected pins

```sh
git log --oneline -2                # integration commit on top of aa8adfe9bf21
git -C KernelSU describe --tags     # v4.2.0-rc3
ls -l drivers/kernelsu              # -> ../KernelSU/kernel
```

## Package the boot image

polaris uses header v1, page size 4096, kernel load offset 0x8000, ramdisk
0x1000000, tags 0x100. Take the ramdisk and cmdline from a stock LineageOS
polaris boot image (the ROM zip, or this release's boot image, whose ramdisk is
the unmodified stock one).

```sh
# one-time: AOSP mkbootimg tools
git clone --depth=1 https://android.googlesource.com/platform/system/tools/mkbootimg

python3 mkbootimg/unpack_bootimg.py --boot_img stock-boot.img --out stock > stock-info.txt
CMDLINE=$(sed -n 's/^command line args: //p' stock-info.txt)

python3 mkbootimg/mkbootimg.py \
  --kernel out/arch/arm64/boot/Image.gz-dtb --ramdisk stock/ramdisk \
  --base 0x0 --kernel_offset 0x8000 --ramdisk_offset 0x1000000 \
  --second_offset 0x0 --tags_offset 0x100 --pagesize 4096 --header_version 1 \
  --os_version 15.0.0 --os_patch_level 2026-11 --cmdline "$CMDLINE" \
  -o boot-lineage-22.2-ksu-susfs.img
truncate -s 67108864 boot-lineage-22.2-ksu-susfs.img   # 64 MiB boot partition
```

`unpack_bootimg.py` writes the extracted files to `--out` (default `./out`,
which is the kernel build directory here) — always pass `--out` as shown.

The image is unsigned; the bootloader is unlocked, so no AVB signing is needed.

Flash: `adb reboot bootloader && fastboot flash boot boot-lineage-22.2-ksu-susfs.img && fastboot reboot`.
Recovery: flash the stock LineageOS polaris boot image with the same command.

## Post-flash verification

```sh
adb shell 'cat /proc/version'                             # -perf-g<sha>, no -dirty
adb shell 'dmesg | grep -i "susfs is initialized"'        # v2.3.0
adb shell '/data/adb/ksu/bin/ksu_susfs show version'      # v2.3.0
adb shell '/data/adb/ksu/bin/ksu_susfs show enabled_features'
adb shell '/data/adb/ksu/bin/ksud --version'
```

Manager: open the APK, expect "Working / Built-in" with driver + manager
version codes equal. Zygisk modules: `cat /data/adb/rezygisk/state.json`
(zygote 32/64 state + loaded module list).

## Notes / pitfalls

- `CONFIG_COMPAT_VDSO=n` in `mix2s_defconfig`: the 32-bit vDSO needs a
  soft-float ARM (armhf) toolchain plus `CROSS_COMPILE_ARM32`; 32-bit apps still
  run without it, so only re-enable if that toolchain is installed.
- A metamodule (meta-overlayfs) is **not** required; modules install and mount
  with ksud's built-in implementation.
- `-dirty` reappears if any tracked file is modified or the submodule is dirty;
  build out-of-tree (`O=out`) and commit before building.
- SUSFS `sus_path`/`sus_map` only hide from unmounted processes with uid >= 10000
  (apps), not from `shell`/root — that is by design.
