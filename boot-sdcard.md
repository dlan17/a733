# Boot mainline u-boot (leverage exist vendor boot0)

This article describe how to deploy mainline u-boot with the help of exist vendor boot0(mainly for DDR initialization).

Example here demonstrate how to boot from storage media (e.g: micro SD card), the method used here can be applied to

other medias, like:  eMMC, SPI flash..



## Preparation

### Build vendor bootloader

Build vendor bootloader using code from Radxa's repository, to get the boot0 image for corresponding storage media,

The boot0 images is responsible for DDR initialization and chain loading second stage images.



```sh
$ export SDK=~/sdkproj
$ mkdir -p ${SDK} && cd ${SDK}
$ git clone --recursive https://github.com/radxa-pkg/u-boot-aw2501.git && cd u-boot-aw2501
$ make install_toolchain
$ sudo apt-get install dos2unix
$ make radxa-cubie-a7a_build
$ make out/radxa-cubie-a7a/boot0_sdcard.bin
```

This will generate `src/u-boot-sun60iw2p1.bin` and `out/radxa-cubie-a7a/boot0_sdcard.bin`.


To flash it into sdcard, using the command as follow:

```sh
$ sudo dd conv=notrunc,fsync if=out/radxa-cubie-a7a/boot0_sdcard.bin of=/dev/sdX bs=512 seek=256
```
(Please replace /dev/sdX to the sdcard device in your host machine)

---

**NOTE**

If you want to boot from SPI flash instead of SD card, you need to run one additional command:

```sh
$ make out/radxa-cubie-a7a/boot0_spinor.bin
```

This will generate `out/radxa-cubie-a7a/boot0_spinor.bin` which is specifically target for SPI NOR flash.

You can then flash it to the SPI flash.

---

### Mainline u-boot and atf

1. mainline u-boot

download u-boot source:
```sh
 $ git clone -b allwinner/A733/boot https://github.com/dlan17/u-boot

```

2. TF-A port for A733

download TF-A source:
```sh
 $ git clone -b A733 https://github.com/dlan17/trusted-firmware-a

```

Please refer to boot-fel.md for how to build them. After compilation,

you should find `u-boot-sunxi-with-spl.bin` and `u-boot-sunxi-with-spl.fit.fit` under the u-boot directory,

Then export the path as below:

```sh
$ export UBOOT=/path/to/u-boot
```
(Please replace /path/to/u-boot to your actual path)

## Pack the image

Enter an empty workspace:

```sh
$ export PAK=~/packproj
$ mkdir ${PAK} && cd ${PAK}
```

Extract the head from vendor u-boot:

```sh
$ dd if=${SDK}/u-boot-aw2501/src/u-boot-sun60iw2p1.bin of=header-info.bin bs=1600 count=1
```

Modify the load address of u-boot-spl to 0x48000000:

> To run mainline u-boot, we need this step to avoid overlap between u-boot-spl and u-boot.
> The default load address from vendor's version in `header-info.bin` is 0x4a000000, which is the same runtime address of mainline u-boot.
> Problem is that if mainline u-boot-spl image is loaded to 0x4a000000, it would be overwritten later when executing the u-boot image.
> So, we have to run following command to fix this:

```sh
$ perl -e 'print pack("V", 0x48000000)' | dd of=${PAK}/header-info.bin bs=1 seek=$((0x2c)) count=4 conv=notrunc
```

Copy the mainline u-boot binary:

```sh
$ cp ${UBOOT}/u-boot-sunxi-with-spl.{bin,fit.fit} ${PAK}
```

Combine head and mainline u-boot into final image:

```sh
$ cat header-info.bin u-boot-sunxi-with-spl.bin > u-boot-sunxi-with-spl-with-head.bin
```

Prepare the pack config file:

> item=dtb will be loaded to base address of item=u-boot (0x48000000) + 2M (0x200000) = 0x48200000

```sh
$ cat > boot_package.cfg << EOF
[package]
item=u-boot, u-boot-sunxi-with-spl-with-head.bin
item=dtb, u-boot-sunxi-with-spl.fit.fit
EOF
```

Before packing,  please check the files in the directory of `${PAK}`, it should resemble as follow:
```
├── boot_package.cfg
├── header-info.bin
├── u-boot-sunxi-with-spl.bin
├── u-boot-sunxi-with-spl.fit.fit
└── u-boot-sunxi-with-spl-with-head.bin
```

Pack the image:
```sh
$ cd ${PAK}
$ ${SDK}/u-boot-aw2501/tools/pack/pctools/linux/openssl/dragonsecboot -pack ./boot_package.cfg
```

This will generate `boot_package.fex`, which can be flashed to micro SD card with below command.

```sh
$ sudo dd conv=notrunc,fsync if=${PAK}/boot_package.fex of=/dev/sdX bs=512 seek=32800
```
