<p align="center"><img src="https://raw.githubusercontent.com/cloud-boot/brand/main/social/cloud-boot.png" alt="cloud-boot/kernel" width="720"></p>

# cloud-boot/kernel

Reproducible end-to-end test scaffolding for [`go-coff/stub`](../stub):
Dockerfiles that produce minimal cloud-guest Linux kernels (PE32+
EFISTUB) and a microscopic initramfs whose `/init` prints a magic
banner before powering the VM down.

`task` from the stub repo then assembles them into a UKI via `pec` and
boots it under QEMU + OVMF to validate that the whole pipeline — stub
chain-load + EFI_LOAD_FILE2_PROTOCOL initrd + kernel handoff — works
on a real Linux kernel.

## Variants

| File                       | Arch  | Approach                                     | Output             |
| -------------------------- | ----- | -------------------------------------------- | ------------------ |
| `Dockerfile.arm64`         | arm64 | `defconfig` + cloud fragment + vmlinuz.efi   | `Image`            |
| `Dockerfile.arm64-tiny`    | arm64 | `tinyconfig` + tiny fragment + vmlinuz.efi   | `Image`            |
| `Dockerfile.arm64-lto`     | arm64 | Clang ThinLTO (reference, no measured win)   | `Image`            |
| `Dockerfile.amd64`         | amd64 | `defconfig` + cloud fragment                 | `bzImage`          |
| `Dockerfile.amd64-tiny`    | amd64 | `tinyconfig` + tiny fragment                 | `bzImage`          |
| `Dockerfile.arm64-disk`    | arm64 | tiny + ext4 + GPT + kexec (bootstrap kernel) | `Image`            |
| `Dockerfile.amd64-disk`    | amd64 | tiny + ext4 + GPT + kexec (bootstrap kernel) | `bzImage`          |
| `Dockerfile.initrd-arm64`  | arm64 | tiny asm `/init` (write banner + reboot)     | `initramfs.cpio.gz` |
| `Dockerfile.initrd-amd64`  | amd64 | same, x86_64 syscall ABI                     | `initramfs.cpio.gz` |

Naming convention: **amd64** and **arm64** everywhere except inside the
Dockerfiles themselves, which still talk to the kernel build system in
its native vocabulary (`ARCH=x86_64`, `ARCH=arm64`) and use the
canonical LLVM/GCC triples (`x86_64-linux-gnu`, etc.).

## Build

```sh
# Tiny + ZBOOT arm64 kernel, ~2.2 MiB.
docker build -f Dockerfile.arm64-tiny -t go-coff-kernel:arm64-tiny .
docker build -f Dockerfile.initrd-arm64 -t go-coff-initrd:arm64 .
```

## Extract

```sh
docker create --name k go-coff-kernel:arm64-tiny && docker cp k:/Image . && docker rm k
docker create --name i go-coff-initrd:arm64      && docker cp i:/initramfs.cpio.gz . && docker rm i
```

## Use

```sh
cd ../stub
go run ../pec append \
    --linux=../kernel/Image \
    --initrd=../kernel/initramfs.cpio.gz \
    --cmdline=<(echo -n "console=ttyAMA0 ip=dhcp") \
    -o BOOTAA64-real.EFI BOOTAA64.EFI
# … then boot under QEMU + OVMF as usual.
```

(The final EFI binary name `BOOTAA64.EFI` is mandated by the UEFI spec
for the removable-media fallback path on arm64 — that's why it doesn't
match our `arm64` convention. Same story for `BOOTX64.EFI` on amd64.)

The first build of any variant takes 5–15 min (Linux source tarball +
native `make`); subsequent builds hit the layer cache and are instant.

## License

[BSD 3-Clause](../stub/LICENSE).
