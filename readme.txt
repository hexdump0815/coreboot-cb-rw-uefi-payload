# these are some notes about creating a rw payload uefi firmware for chromebooks

note: further down there is a maybe simpler and better approach than the one described here first

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

- steps to build it on a debian x86_64 machine:

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

- result: build/UEFIPAYLOAD.fd

- transfer result file over to chromebook

- on chromebook:
# save a backup copy of the original bios image before changing anything
# always good to have around so better transfer it to a safe place ...
flashrom -p host -r bios.bin.backup
cp bios.bin.backup bios.bin
# remove the original altfw tianocore payload
cbfstool bios.bin remove -r RW_LEGACY -n altfw/tianocore
# i guess this is required to avoid chromeos updates overwriting our changed firmware
cbfstool bios.bin remove -r RW_LEGACY -n cros_allow_auto_update
# add the new altfw tianocore payload
cbfstool bios.bin add-payload -r RW_LEGACY -n altfw/tianocore -f UEFIPAYLOAD.fd -c lzma
# write the RW_LEGACY section of the adjusted firmware to flash
flashrom -p host -i RW_LEGACY -w bios.bin
# enable altfw booting and make it the default (might not work if not set to default)
crossystem dev_boot_altfw=1
crossystem dev_default_boot=altfw
#reboot with an uefi linux image on usb or sd card


there is a maybe simpler (as nothing needs to be compiled) and safer (as it
is using tested uefi payloads from mrchromebox as base) approach which is
inspired by https://github.com/MrChromebox/firmware/issues/122#issuecomment-527243479
this approach is for creating a RW_LEGACY style firmware file, but it should
be possible to apply the same basic idea also to the altfw case (described
below the RW_LEGACY notes) ... the below notes use an apollo lake asus c423
chromebook (rabbid) as an example

- first find the codename of your chromebook and check if there is a uefi firmware for it at https://mrchromebox.tech/#devices
- next find the name of the firmware file at https://github.com/MrChromebox/scripts/blob/master/sources.sh
- on chromebook:
# save a backup copy of the original bios image before changing anything
# always good to have around so better transfer it to a safe place ...
flashrom -p host -r bios.bin.backup
cp bios.bin.backup bios.bin
# double check the size of the RW_LEGACY section - required below
fmap_decode bios.bin | grep RW_LEGACY
# get the mrchromebox uefi firmware - file should be adjusted to chromebook type
curl -LO https://www.mrchromebox.tech/files/firmware/full_rom/coreboot_tiano-rabbid-mrchromebox_20220718.rom
# extract the uefi payload from it
cbfstool coreboot_tiano-rabbid-mrchromebox_20220718.rom extract -n fallback/payload -m x86 -f cbox.pl
# create a proper RW_LEGACY firmware file out of it - keep in mind the side found out above
cbfstool rwl.bin create -m x86 -s 0x00200000
cbfstool rwl.bin add-payload -f cbox.pl -c lzma -n payload
dd if=/dev/zero of=smmstore bs=256k count=1
util/cbfstool/cbfstool rwl.bin add -f smmstore -n "smm store" -t raw -b 0x1bf000
# write the freshly created rw uefi firmware to the RW_LEGACY section of the flash
flashrom -p host -w -i RW_LEGACY:rwl.bin
# enable usb booting and make it the default (if wanted - otherwise: ctrl-u for usb and and ctrl-d for emmc on boot)
crossystem dev_boot_usb=1
crossystem dev_default_boot=usb
#reboot with an uefi linux image on usb or sd card


this is the procedure if altfw is supported - example: hp chromebook 14a (blooglet)

- first find the codename of your chromebook and check if there is a uefi firmware for it at https://mrchromebox.tech/#devices
- next find the name of the firmware file at https://github.com/MrChromebox/scripts/blob/master/sources.sh
- on chromebook:
# save a backup copy of the original bios image before changing anything
# always good to have around so better transfer it to a safe place ...
flashrom -p host -r bios.bin.backup
cp bios.bin.backup bios.bin
# get the mrchromebox uefi firmware - file should be adjusted to chromebook type
curl -LO https://www.mrchromebox.tech/files/firmware/full_rom/coreboot_tiano-blooglet-mrchromebox_20220718.rom
# extract the uefi payload from it
cbfstool coreboot_tiano-blooglet-mrchromebox_20220718.rom extract -n fallback/payload -m x86 -f cbox.pl
# remove the original altfw tianocore payload
cbfstool bios.bin remove -r RW_LEGACY -n altfw/tianocore
# i guess this is required to avoid chromeos updates overwriting our changed firmware
cbfstool bios.bin remove -r RW_LEGACY -n cros_allow_auto_update
# add the new altfw tianocore payload
cbfstool bios.bin add-payload -r RW_LEGACY -n altfw/tianocore -f cbox.pl -c lzma
# write the RW_LEGACY section of the adjusted firmware to flash
flashrom -p host -i RW_LEGACY -w bios.bin
# enable altfw booting and make it the default (might not work if not set to default)
crossystem dev_boot_altfw=1
crossystem dev_default_boot=altfw
#reboot with an uefi linux image on usb or sd card
