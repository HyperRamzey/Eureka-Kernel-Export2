# Eureka-Kernel R24U — CIP114 Upstream + KernelSU (Clang 24 / -O2 / ThinLTO / ICF)

Custom kernel for Samsung Galaxy Exynos7885 devices, based on
[Eureka-Kernel](https://github.com/eurekadevelopment/Eureka-Kernel) R24U with:

- **Linux 4.4.302-cip114** — the kernel base is upstreamed from Samsung's 4.4.302
  vendor drop to the final mainline 4.4.x stable, plus **114 CIP (Civil Infrastructure
  Platform) SLTS maintenance releases** (528 cherry-picked stable/security fixes +
  hand-merged vendor-conflicted hunks: cred refcount hardening, f2fs fsync-recovery,
  ext4/f2fs/i2c/ufs/mm/net fixes, USB serial device IDs, tcp/ipv6 WRITE_ONCE, etc.)
- **KernelSU (rsuntk fork, RKSU v3.2.2-10-legacy / KSU v32473)** — kernel-level root
  with manual hooks (execveat/faccessat/stat/vfs_read/input/selinux) pre-integrated
  in this tree, plus the `path_umount` backport (Linux 5.9) so KSU module unmount
  works on 4.4.
- **All-Clang/LLVM build**: Neutron Clang `24.0.0git`, `-O2`
  (`CONFIG_CC_OPTIMIZE_FOR_PERFORMANCE` — stock O3/Ofast disabled),
  **ThinLTO** (`CONFIG_LTO_CLANG` + `CONFIG_THINLTO`) linked with `ld.lld`,
  **`--icf=all`** (identical code folding, final link only), and
  `-march=armv8-a+crc+crypto` (verified against the A30s Cortex-A53/A73
  hardware features: fp asimd aes pmull sha1 sha2 crc32).

## Supported devices

| Device | Model | Codename |
| --- | --- | --- |
| Galaxy A10 | SM-A105F | a10 |
| Galaxy A20 | SM-A205F | a20 |
| Galaxy A20e | SM-A202F | a20e |
| Galaxy A30 | SM-A305F | a30 |
| Galaxy A30s | SM-A307FN | a30s |
| Galaxy A40 | SM-A405FN | a40 |
| Galaxy M20 | SM-M205F | m20 |
| Galaxy A2 Core | SM-A260F | jackpotlte |

## Installation

1. Unlock bootloader, install a custom recovery (TWRP) for your device.
2. Download `Eureka-R24U-CIP114-KSU-O2-ThinLTO-ICF_<device>.zip` from
   [Releases](../../releases).
3. Boot into recovery, flash the zip (AnyKernel3 — it flashes boot + dtbo).
4. Optional: after first boot, install the RKSU manager APK from the
   [rsuntk/KernelSU releases](https://github.com/rsuntk/KernelSU/releases)
   (v3.2.2-10-legacy or newer).

> KernelSU is compiled in (`CONFIG_KSU=y`); root is granted per-app through the manager.

## Compilation

Requirements: Linux x86_64 (Debian 13 / Ubuntu 24.04+), ~50 GB disk, ~32 GB RAM for ThinLTO.

### Toolchain

This kernel is built with [Neutron Clang](https://github.com/Neutron-Toolchains/clang-build-catalogue)
(24.0.0git, bare LLVM: clang + ld.lld + llvm-*), plus GNU cross binutils for
`CROSS_COMPILE` helper tools:

```sh
# Debian/Ubuntu
sudo apt install git bc bison flex build-essential cpio zstd \
  gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu \
  gcc-arm-linux-gnueabi binutils-arm-linux-gnueabi device-tree-compiler
```

### Build one device

```sh
export TC=/path/to/neutron-clang   # dir with bin/ lib/
export PATH="$TC/bin:$PATH"
export LD_LIBRARY_PATH="$TC/lib"
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
export CROSS_COMPILE_ARM32=arm-linux-gnueabi-
export CLANG_TRIPLE=aarch64-linux-gnu-
export ANDROID_MAJOR_VERSION=r
export KBUILD_BUILD_USER=you
export LOCALVERSION=-R24U-CIP114-KSU
echo > .scmversion                     # keep utsrelease under 64 chars

make O=out ARCH=arm64 ANDROID_MAJOR_VERSION=r exynos7885-a30s_defconfig
scripts/config --file out/.config \
    -d CC_OPTIMIZE_FOR_PERFORMANCE_O3_OFAST_SUB_OPTIONS \
    -e CC_OPTIMIZE_FOR_PERFORMANCE -e LTO_CLANG -e THINLTO -e KSU
make O=out -j8 ARCH=arm64 ANDROID_MAJOR_VERSION=r LLVM=1 \
    CC=clang HOSTCC=clang HOSTCXX=clang++ \
    LD=ld.lld AR=llvm-ar NM=llvm-nm STRIP=llvm-strip OBJDUMP=llvm-objdump \
    KCFLAGS="-march=armv8-a+crc+crypto"
```

Outputs: `out/arch/arm64/boot/{Image,dtb.img,dtbo.img}`. Package with the
AnyKernel3 template in `kernel_zip/anykernel/` (copy Image + dtb.img + the
prebuilt device dtbo from `kernel_zip/dtbo/<device>/`, zip `META-INF tools
anykernel.sh Image dtb.img dtbo.img version`).

### Verification flags (as used for the shipped release)

- `-O2`: `CONFIG_CC_OPTIMIZE_FOR_PERFORMANCE=y`, O3/Ofast unset
- ThinLTO: `CONFIG_LTO_CLANG=y`, `CONFIG_THINLTO=y`, `--thinlto-cache-dir` in the vmlinux link
- ICF: `--icf=all` in `LDFLAGS_vmlinux` (final link only; lld forbids it with `ld -r`)
- march: `-march=armv8-a+crc+crypto` in `KCFLAGS`

## Source layout / credits

- Base: Eureka-Kernel R24U (eurekadevelopment) — Samsung Exynos7885 BSP, OneUI/Q/R/S support
- Upstream: [CIP linux-4.4.y SLTS](https://gitlab.com/cip-project/cip-kernel/linux-cip) (v4.4.302-cip114)
- Root: [rsuntk/KernelSU](https://github.com/rsuntk/KernelSU) (RKSU, non-GKI 4.4-supported fork)
- Packaging: AnyKernel3

Branch `eureka-r24u-cip114-ksu` contains the full history: base snapshot →
CIP114 merge → per-fix commits → KernelSU port → build fixes.
