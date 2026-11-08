# Prepare 
- Install Litex
- Update boards files from litex-boards

# Create SoC
- RV32
```shell
mkdir -p workarea; cd workarea;
python3 -m litex_boards.targets.qmtech_kintex7_devboard --build \
--bus-standard axi \
--cpu-type vexiiriscv --cpu-variant=linux --cpu-count=1 \
--with-video-framebuffer --with-sdcard --with-usb \
--with-ethernet --with-coherent-dma \
--uart-baudrate 115200 --uart-name gpio_serial
cd ..
```
- RV64
```shell
mkdir -p workarea; cd workarea;
python3 -m litex_boards.targets.qmtech_kintex7_devboard --build \
--bus-standard axi \
--cpu-type vexiiriscv --cpu-variant=debian --cpu-count=2 \
--with-video-framebuffer --with-sdcard --with-usb \
--with-ethernet --with-coherent-dma --l2-byte=262144 \
--uart-baudrate 115200 --uart-name gpio_serial
cd ..
```

# Terminal
```shell
sudo chown alex:alex /dev/ttyUSB0
cu -l /dev/ttyUSB0 -s 115200 or litex_term /dev/ttyUSB0;
```

# Create dts
```shell
cd workarea;
python3 -m litex.tools.litex_json2dts_linux build/qmtech_kintex7_devboard/csr.json > ../linux.dts
dtc -O dtb -I dts -o ../images/linux.dtb ../linux.dts
cd ..
```

# Toolchain
```shell
mkdir -p tools; cd tools;
wget https://github.com/xpack-dev-tools/riscv-none-elf-gcc-xpack/releases/download/v12.3.0-2/xpack-riscv-none-elf-gcc-12.3.0-2-linux-x64.tar.gz
tar -xvzf xpack-riscv-none-elf-gcc-12.3.0-2-linux-x64.tar.gz
wget https://github.com/riscv-collab/riscv-gnu-toolchain/releases/download/2024.04.12/riscv64-glibc-ubuntu-22.04-gcc-nightly-2024.04.12-nightly.tar.gz
tar -xvzf riscv64-glibc-ubuntu-22.04-gcc-nightly-2024.04.12-nightly.tar.gz
cd ..
```

# OpenSBI
OpenSBI without payload and with DT
- RV32
```shell
git clone https://github.com/Dolu1990/opensbi.git opensbi_32 --branch upstream
cd opensbi_32
make clean
make CROSS_COMPILE=../tools/riscv/bin/riscv64-unknown-linux-gnu- \
     PLATFORM=generic \
     PLATFORM_RISCV_XLEN=32 \
     PLATFORM_RISCV_ISA=rv32ima_zicsr_zifencei \
     PLATFORM_RISCV_ABI=ilp32 \
     FW_FDT_PATH=../images/linux.dtb \
     FW_JUMP_ADDR=0x40000000  \
     FW_JUMP_FDT_ADDR=0x40ef0000 \
     -j$(nproc)
cp build/platform/generic/firmware/fw_jump.bin ../images/opensbi.bin
cd ..
```
- RV64
```shell
git clone https://github.com/riscv-software-src/opensbi.git
cd opensbi
make clean
make CROSS_COMPILE=../tools/riscv/bin/riscv64-unknown-linux-gnu- \
     PLATFORM=generic \
     PLATFORM_RISCV_XLEN=64 \
     PLATFORM_RISCV_ISA=rv64gc_zicsr_zifencei \
     PLATFORM_RISCV_ABI=lp64d \
     FW_FDT_PATH=../images/linux.dtb \
     FW_JUMP_ADDR=0x40000000  \
     FW_JUMP_FDT_ADDR=0x40ef0000 \
     -j$(nproc)
cp build/platform/generic/firmware/fw_jump.bin ../images/opensbi.bin
cd ..
```

<!-- # Linux kernel
```shell
git clone git@github.com:Dolu1990/litex-linux.git
cd litex-linux
git checkout f5ee078b
cp ../linux.config .config
export CROSS_COMPILE=../tools/riscv/bin/riscv64-unknown-linux-gnu-
make -j$(nproc) ARCH=riscv oldconfig
make -j$(nproc) ARCH=riscv all
ls vmlinux               # Kernel elf (has symboles, usefull for debug)
ls arch/riscv/boot/Image # Kernel binary
cp ./arch/riscv/boot/Image ../images/Image
cd ..
unset CROSS_COMPILE
``` -->

# Buildroot (Kernel + rootFS)
Check rootfs size after build and correct initrd in DT.
To enable display data in monitor, add to bootargs console=tty0.
If you want to disable start gui, remove console=tty0 from DT and add empty file to `buildroot_configs_64_gui/board/litex_soc/rootfs_overlay/etc/init.d/S99xdm`
- RV32
```shell
git clone git://git.busybox.net/buildroot
cd buildroot; git checkout 2025.02.10;
export CROSS_COMPILE=../tools/riscv/bin/riscv64-unknown-linux-gnu-
make O=build/basic  BR2_EXTERNAL=../buildroot_configs_32 litex_soc_basic_defconfig
(cd build/basic/ && make -j$(nproc))
unset CROSS_COMPILE
cp ./build/basic/images/rootfs.cpio ../images/rootfs.cpio
cp ./build/basic/images/Image ../images/Image
```
- RV64
```shell
git clone git://git.busybox.net/buildroot
cd buildroot; git checkout 2025.02.10;
export CROSS_COMPILE=../tools/riscv/bin/riscv64-unknown-linux-gnu-
make O=build/basic  BR2_EXTERNAL=../buildroot_configs_64_gui litex_soc_basic_defconfig
(cd build/basic/ && make -j$(nproc))
unset CROSS_COMPILE
cp ./build/basic/images/rootfs.cpio ../images/rootfs.cpio
cp ./build/basic/images/Image ../images/Image
```
# Prepare SD
```shell
TODO
```