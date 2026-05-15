# go-coff/kernel-build

Reproducible end-to-end test scaffolding for [`go-coff/stub`](../stub):
two Dockerfiles that produce, respectively, a minimal aarch64 Linux
kernel (PE32+ EFISTUB) and a microscopic initramfs whose `/init` prints
a magic banner before powering the VM down.

`task` from the stub repo then assembles them into a UKI via `pec` and
boots it under QEMU + OVMF to validate that the whole pipeline — stub
chain-load + EFI_LOAD_FILE2_PROTOCOL initrd + kernel handoff — works
on a real Linux kernel.

## Build

```sh
docker build -f Dockerfile.aa64   -t go-coff-kernel:aa64  .
docker build -f Dockerfile.initrd -t go-coff-initrd:aa64  .
```

## Extract

```sh
docker create --name k go-coff-kernel:aa64  && docker cp k:/Image . && docker rm k
docker create --name i go-coff-initrd:aa64  && docker cp i:/initramfs.cpio.gz . && docker rm i
```

## Use

```sh
cd ../stub
go run ../pec --add-section .linux=../kernel-build/Image \
              --add-section .initrd=../kernel-build/initramfs.cpio.gz \
              --add-section .cmdline=<(echo -n "console=ttyAMA0") \
              -o BOOTAA64-real.EFI BOOTAA64.EFI
# … then boot under QEMU + OVMF as usual.
```

The first build of `Dockerfile.aa64` takes ~5–15 min depending on the
host (it pulls the Linux source tarball and runs `make defconfig &&
make Image` natively under the aarch64 Docker VM); subsequent builds
hit the cache and are instant.

## License

[BSD 3-Clause](../stub/LICENSE).
