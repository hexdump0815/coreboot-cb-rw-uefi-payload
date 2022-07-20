# these are some notes about creating a rw payload uefi firmware for chromebooks

based on:
https://github.com/MrChromebox/scripts/issues/89#issuecomment-1181326697

with the some more info from:
https://www.coreboot.org/Build_HOWTO
https://medium.com/swlh/getting-started-with-coreboot-7ade78327e75

tested on debian bullseye on an x86_64 machine building the payload
for octopus-blooglet (should actually work for all octopus systems)

advantage: it is possible to boot regular uefi linux images with a
firmware which does not require full rw access to the chromebook
firmware flash - only the rw accessable part of it needs to be
rewritten for it

more info coming soon ...

steps to build it on a debian x86_64 machine:
# imagemagick is only required in case a custom bootsplash.bmp is used - the file needs to be in the coreboot dir then
apt-get -y install git build-essential gnat flex bison libncurses5-dev wget zlib1g-dev uuid-dev nasm python2.7 imagemagick python3-distutils python3-apt
git clone https://review.coreboot.org/coreboot
git checkout 4.17
git submodule update --init --checkout
make crossgcc-i386 CPUS=2
make menuconfig
# mainboard -> vendor: google - board: octopus
# payload -> tianocore -> tianocore bootsplash path and filename: make the field empty
# exit -> save
# 
# check if a python binary is already there, otherwise link it
ls -l /usr/bin/python
ln -s /usr/bin/python2.7 /usr/bin/python
# ignore the fsp and ifd warning at the end of the build - not relevant as we will only use the uefi payload
make
# remove the python link in case it was created above
rm -i /usr/bin/python

result: build/UEFIPAYLOAD.fd

transfer result file over to chromebook

on chromebook:
# save a backup copy of the original bios image before changing anything
# always good to have around so better transfer it to a safe place ...
flashrom -p host -r bios.bin.backup 
flashrom -p host -r bios.bin
cbfstool bios.bin remove -r RW_LEGACY -n altfw/tianocore
cbfstool bios.bin remove -r RW_LEGACY -n cros_allow_auto_update
cbfstool bios.bin add-payload -r RW_LEGACY -n altfw/tianocore -f UEFIPAYLOAD.fd -c lzma
flashrom -p host -i RW_LEGACY -w bios.bin

crossystem dev_boot_altfw=1
crossystem dev_default_boot=altfw

reboot with an uefi linux image on usb or sd card
