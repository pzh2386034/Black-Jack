
nmap -sn 192.168.1.0/24

sudo nano /boot/config.txt

    add:
        dtoverlay=pi3-miniuart-bt-overlay
        force_turbo=1


## close hciuart

sudo systemctl disable hciuart


## dhcp server

sudo docker run -it --rm --init --net host -v "/etc/dhcp":/data networkboot/dhcpd enx081f7135c299 &

1. sudo apt install isc-dhcp-server
2. configure /etc/dhcp/dhcpd.conf


## enable qemu

* Get QEMU v2.12.0 for -M raspi3

``` bash
git clone git://git.qemu.org/qemu.git
cd qemu
git checkout v2.12.0
./configure --target-list=aarch64-softmmu
make -j`nproc`
export PATH="$(pwd)/build/aarch64-softmmu:${PATH}"
cd ..

# Get source.
git clone https://github.com/bztsrc/raspi3-tutorial
cd raspi3-tutorial
git checkout efdae4e4b23c5b0eb96292f2384dfc8b5bc87538

# Setup to use the host precompiled cross compiler.
# https://github.com/bztsrc/raspi3-tutorial/issues/30
sudo apt-get install gcc-aarch64-linux-gnu
find . -name Makefile | xargs sed -i 's/aarch64-elf-/aarch64-linux-gnu-/'

# Compile and run stuff.
cd 05_uart0
make
../../qemu/aarch64-softmmu/qemu-system-aarch64 -M raspi3 -kernel kernel8.img -serial stdio
```

* note

    1. QEMU doesn't support config.txt, so we need to follow the default booting procedure.
        The kernel is loaded to address 0x80000 instead of 0x0. Simply modifying linker.ld should work.
        The processor runs at EL2 instead of EL3. boot.S should be adapted to configure spsr_el2 and elr_el2.
        Make an identify mapping to avoid "prefetch abort".

    2. QEMU only supports UART0 (PL011), UART1 (mini UART) is not sent to stdio.

    3. QEMU doesn't emulate the system timer. I use ARM generic timer to generate timer interrupts.

## 参考资料

[ARM setup lab](http://www.bravegnu.org/gnu-eprog/index.html#intro)

[raspberry-pi-os](https://github.com/s-matyukevich/raspberry-pi-os)

[raspi3-tutorial](https://github.com/bztsrc/raspi3-tutorial)