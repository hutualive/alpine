# background
alpine linux is popular in docker, but the simple, small design also promising for embedded world. so I'm giving it a try to bootstrap to stm32mp1 discovery kit.
the main drawback compare to other distributions is less documents, so hope here can help you a little bit to run alpine on armv7 soc like stm32mp1.

# quick start guide
in the end of process, you will have a complete sd card image, dd to the sd card, you can boot into alpine 3.10 as root without password.

sudo dd if=./alpine-sd-dk2.img of=/dev/sdb bs=8M conv=fdatasync

//caution: check carefully your device file name, here for my case is /dev/sdb for my usb sd card reader.

# introduction
st use GPT(GUID partition table)as partition mechanisim. the rom bootloader inside the soc will look for the partition name fsbl1 for first stage bootloader. if failed, will look for fsbl2 as backup. st choose to use arm tf-a for fsbl1 and fsbl2. first stage bootloader will load second stage loader(ssbl), here is u-boot which is popular in embedded world. then second stage bootloader will load linux kernel and device tree blob. these are soc/board specific parts.

alpine also use initramfs and modloop to support diskless mode installation, with these infrastructures put in place, you are ready to go :)

so below is the process step by step.

# populate a raw sd card image

create a 128MB raw sd card image file -->

sudo dd if=/dev/zero of=./alpine-sd-dk2.img bs=1M count=128

use gdisk to create the partitions: fsbl1, fsbl2, ssbl, alpine -->

sudo gdisk ./alpine-sd-dk2.img

GPT reserve the first 17KB on sd card for protective MBR and 128 entries of GPT table. so you can only start with 17KB on-wards and you need adjust the sector alignment to 1 as gdisk default is 2048.

you need mark the alpine partition as legacy bios bootable for u-boot to look for kernel and device tree.

below is the summary of partitions and filesystem type:

| size | name | type |
| :----: | :----: | :----: |
| 0-17K: | protective MBR and GPT table |
| 17K+256K: | fsbl1 | linux reserved 8301 |
| 17K+256K+256K: | fsbl2 | linux reserved 8301 |
| 17K+256K+256K+2M: | ssbl | linux reserved 8301 |
| 17K+256K+256K+2M+rest: | alpine | linux filesystem 8300 with legacy bios bootable attribute |

# mount sd card image as disk to populate

sudo losetup -Pf alpine-sd-dk2.img

//ps: here in my case the image file mounted as /dev/loop2 with 4 partitions  

# populate fsbl

inside the fsbl directory, I uploaded st official tf-a firmware for dk2 for your convenience:

sudo dd if=./fsbl/tf-a-dk2.stm32 of=/dev/loop2p1 bs=1M conv=fdatasync  

sudo dd if=./fsbl/tf-a-dk2.stm32 of=/dev/loop2p2 bs=1M conv=fdatasync

# populate ssbl

inside the ssbl directory, I uploaded u-boot firmware for dk2 for your convenience: 

sudo dd if=./ssbl/u-boot-dk2.stm32 of=/dev/loop2p3 bs=1M conv=fdatasync

# populate boot and root file system

inside the alpine directory, I uploaded the kernel, device tree, initramfs, modloop, apks for dk2 for your convenience:

apks is extracted from the generic arm package downloaded from https://www.alpinelinux.org/downloads/ under armv7 

sudo mkfs.ext4 -L alpine /dev/loop2p4

sudo mount -t ext4 /dev/loop2p4 /mnt

rsync -avx ./alpine/ /mnt/

sudo umount /mnt

# clean up

sudo losetup -d /dev/loop2

# wrap up

go back to quick start guide to flash your image

have fun :)
