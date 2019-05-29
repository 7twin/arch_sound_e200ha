# Kernel 5.1

Previous and future kernel versions are available as git-branches. These are the manual instructions, they will be accompanied by a docker based guided script in the near future - to automate most if not all of those instructions here.

# Sound on e200ha with arch

Big thanks to [heikomat](https://github.com/heikomat) - for keeping all the upcoming kernels patched for sound to work and [garchymede](https://github.com/garchymede/archlinux_on_asus_E200HA)'s repository for getting me into the right direction.

## Instructions
The instructions will be split into "server" and "e200ha", it is highly recommended to do the server part either on another computer (preferably running arch [64 bit], because compiling on e.g. ubuntu seemed to cause issues), a virtual machine running arch or a server with arch, because compiling on the e200ha will take several hours compared to just under ~30 minutes. (your time might vary)

Big parts of this tutorial are also directly taken from the [arch wiki](https://wiki.archlinux.org/index.php/Kernels/Traditional_compilation#Compile_the_kernel), so for more explanations and possibly missing notes/dependencies, it is highly advisable to check it out.

**Note:** Before you configure the kernel (read as `Step 3 in [Server]`), you need to copy over the e200ha config file, so your kernel does not include every single option you don't need, to do that you want to execute:
`zcat /proc/config.gz > .config` on your actual e200ha and then transfer that `.config` file to your server or PC compiling the kernel (directly into the root of the cloned `linux` folder)

### Server

1. Execute `git clone --branch cx2072x https://github.com/heikomat/linux.git` to get the newest kernel files provided by heikomat
2. `cd` into the now cloned folder: `cd linux`
3. Do `make clean && make menuconfig`
4. Follow the instructions and activate the items that are described [here](https://github.com/heikomat/linux/blob/cx2072x/cx2072x_fixes_and_manual/building_the_kernel.md#configuring) as of today those are:
	- Enable these configurations:
		- `Device Drivers -> Sound card support -> Advanced Linux Sound Architecture -> ALSA for SoC audio support -> CODEC drivers -> Conexant CX2072 CODEC`
		- `Device Drivers -> Sound card support -> Advanced Linux Sound Architecture -> ALSA for SoC audio support -> Intel ASoC SST drivers -> Intel Machine drivers > Baytrail and Cherrytrail with CX2072X codec`
	- Remove the string in this configuration:
		- `Cryptographic API -> Certificates for signature checking -> Provide system-wide ring of trusted keys -> Additional X.509 keys for default system keyring`
5. (Optional) if you are running this on a server, it is highly recommended to run this in a `screen` session, do that by executing `screen` and then following the next steps, once you re-connect to your server you can access the same terminal by doing `screen -R`, to detach from the session, you can press `ctrl+a` then pressing `d` or by just closing your ssh client
6. Now compile the kernel by executing: `make` preferably with: `make -j 8` where `8` is the amount of cores your machine has - it'll be much faster the more cores you can assign to it

After it is finished (it'll stop outputting further info and will allow you to enter commands again) download/transfer the `linux` folder to e.g. some usb stick, you can zip it up to save some time by executing `zip -9 -r linux.zip linux/` outside the linux folder, remember to unzip it on your usb-stick later on.

**Note:** If the install fails, look closely at what it says, the errors are very descriptive and can easily tell you what is missing or has to be done. (e.g. on windows unarchiving the zip file sometimes can lead to warnings and not unpack all things in the `/net` folder, so then you would just drag & drop the net folder into the unarchived space and just press `skip` when it asks you to overwrite existing files, so it only extracts what is missing)

### e200ha

1. Navigate to the `linux` folder you transferred from the other computer/server
2. Execute `sudo su` to change to the root user as advised by the arch wiki
3. Now execute `make modules_install` it will install all that is needed from the `linux` folder
4. After it is finished; assuming you are running 64-bit (else the arch-wiki got you covered); execute `cp -v arch/x86_64/boot/bzImage /boot/vmlinuz-linux48` (`linux48` can be named anything you would like in the current and all following steps [e.g. I use `linux420sound` to indicate what kernel it is and that it has sound patched in], just make sure you only change `linux48` and nothing before or after it)
5. Copy the preset `cp /etc/mkinitcpio.d/linux.preset /etc/mkinitcpio.d/linux48.preset` and then edit `/etc/mkinitcpio.d/linux48.preset` values:
	- `ALL_kver` to `ALL_kver="/boot/vmlinuz-linux48"`
	- `default_image` to `default_image="/boot/initramfs-linux48.img"`
	- `fallback_image` to `fallback_image="/boot/initramfs-linux48-fallback.img"`
6. Now run `mkinitcpio -p linux48` to generate the initramfs images
7. Assuming you are using GRUB as your bootloader, do: `grub-mkconfig -o /boot/grub/grub.cfg` it'll auto-detect the new kernel and add it to the boot-menu
8. To customize the boot-order and other things, it is handy to use: `grub-customizer` available in the [default packages](https://www.archlinux.org/packages/community/x86_64/grub-customizer/). (install via `pacman -S grub-customizer` and then just open it or execute `grub-customizer` in your terminal)
9. Install `pulseaudio` if you don't yet have it: `pacman -S pulseaudio`
10. Next steps are taken from the previously mentioned [page](https://github.com/heikomat/linux/tree/cx2072x/cx2072x_fixes_and_manual) too; Get the needed config files:

```
sudo mkdir --parents /usr/share/alsa/ucm/bytcht-cx2072x
cd /usr/share/alsa/ucm/bytcht-cx2072x
sudo wget "https://raw.githubusercontent.com/heikomat/linux/cx2072x/cx2072x_fixes_and_manual/bytcht-cx2072x/HiFi.conf"
sudo wget "https://raw.githubusercontent.com/heikomat/linux/cx2072x/cx2072x_fixes_and_manual/bytcht-cx2072x/bytcht-cx2072x.conf"
```

11. Set `realtime-scheduling` to `realtime-scheduling = no` in `/etc/pulse/daemon.conf`
12. Reboot
