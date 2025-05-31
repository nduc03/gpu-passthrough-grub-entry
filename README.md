## Create custom GRUB entry for NVIDIA GPU passthrough

Tested and worked on Ubuntu 25.04 with AMD system

May create a script to automate these steps in the future.

### Step 1: Find and Copy Your GRUB Menu Entry to Customize
Find default menuentry:

`sudo grep -A20 "menuentry '<distro_name>'" /boot/grub/grub.cfg`

then copy that menuentry for the next step.


Example:

`sudo grep -A20 "menuentry 'Ubuntu'" /boot/grub/grub.cfg`

Example output:
```sh
menuentry 'Ubuntu' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-e98e3d8d-a0b5-4d9a-80db-da976477c490' {
	recordfail
	load_video
	gfxmode $linux_gfx_mode
	insmod gzio
	if [ x$grub_platform = xxen ]; then insmod xzio; insmod lzopio; fi
	insmod part_gpt
	insmod ext2
	set root='hd1,gpt2'
	if [ x$feature_platform_search_hint = xy ]; then
	search --no-floppy --fs-uuid --set=root --hint-bios=hd0,gpt2 --hint-efi=hd0,gpt2 --hint-baremetal=ahci0,gpt2 --hint='hd1,gpt2'  e98e3d8d-a0b5-4d9a-80db-da976477c490
	else
	search --no-floppy --fs-uuid --set=root e98e3d8d-a0b5-4d9a-80db-da976477c490
	fi
	linux	/boot/vmlinuz-6.14.0-15-generic root=UUID=e98e3d8d-a0b5-4d9a-80db-da976477c490 ro  quiet splash crashkernel=2G-4G:320M,4G-32G:512M,32G-64G:1024M,64G-128G:2048M,128G-:4096M $vt_handoff
	initrd	/boot/initrd.img-6.14.0-15-generic
}
submenu 'Advanced options for Ubuntu' $menuentry_id_option 'gnulinux-advanced-e98e3d8d-a0b5-4d9a-80db-da976477c490' {
	menuentry 'Ubuntu, with Linux 6.14.0-15-generic' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-6.14.0-15-generic-advanced-e98e3d8d-a0b5-4d9a-80db-da976477c490' {
		recordfail
		load_video
```
In this example, copy the section starting with `menuentry 'Ubuntu'`


### Step 2: Create a custom GRUB entry
1. Append to `/etc/grub.d/40_custom` the menuentry copied from step 1.

	Example of 40_custom after append:
	```sh
	#!/bin/sh
	exec tail -n +3 $0
	# This file provides an easy way to add custom menu entries.  Simply type the
	# menu entries you want to add after this comment.  Be careful not to change
	# the 'exec tail' line above.
	menuentry 'Ubuntu' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-e98e3d8d-a0b5-4d9a-80db-da976477c490' {
		recordfail
		load_video
		gfxmode $linux_gfx_mode
		insmod gzio
		if [ x$grub_platform = xxen ]; then insmod xzio; insmod lzopio; fi
		insmod part_gpt
		insmod ext2
		set root='hd1,gpt2'
		if [ x$feature_platform_search_hint = xy ]; then
		search --no-floppy --fs-uuid --set=root --hint-bios=hd0,gpt2 --hint-efi=hd0,gpt2 --hint-baremetal=ahci0,gpt2 --hint='hd1,gpt2'  e98e3d8d-a0b5-4d9a-80db-da976477c490
		else
		search --no-floppy --fs-uuid --set=root e98e3d8d-a0b5-4d9a-80db-da976477c490
		fi
		linux	/boot/vmlinuz-6.14.0-15-generic root=UUID=e98e3d8d-a0b5-4d9a-80db-da976477c490 ro  quiet splash crashkernel=2G-4G:320M,4G-32G:512M,32G-64G:1024M,64G-128G:2048M,128G-:4096M $vt_handoff
		initrd	/boot/initrd.img-6.14.0-15-generic
	}
	```
	
2. (Optional but recommended) Rename the Menu Entry:
	- Modify entry name if you want.
	- Example: `menuentry 'Ubuntu (NVIDIA KVM passthrough)'`
	
3. Add VFIO Kernel Parameters
	- Find `linux` command then add these parameters after `quiet splash`: 
	
		`modprobe.blacklist=nouveau,nvidia,nvidia_drm,nvidia_uvm,nvidia_modeset,nvidia_fb use_vfio`
	
	- Example:

		- Before adding parameters:

			```
			linux /boot/vmlinuz-6.14.0-15-generic root=UUID=e98e3d8d-a0b5-4d9a-80db-da976477c490 ro quiet splash crashkernel=2G-4G:320M,4G-32G:512M,32G-64G:1024M,64G-128G:2048M,128G-:4096M $vt_handoff
			```
		- After adding parameters:
			```
			linux /boot/vmlinuz-6.14.0-15-generic root=UUID=e98e3d8d-a0b5-4d9a-80db-da976477c490 ro  quiet splash modprobe.blacklist=nouveau,nvidia,nvidia_drm,nvidia_uvm,nvidia_modeset,nvidia_fb use_vfio crashkernel=2G-4G:320M,4G-32G:512M,32G-64G:1024M,64G-128G:2048M,128G-:4096M $vt_handoff
			```
4. Bonus (Tested on ubuntu):
- Remove kernel version of `vmlinuz-<kernel_version>` and `initrd.img-<kernel_version>` to `vmlinuz`, and `initrd.img` so the script don't have to modify for every kernel update.
- Example:

	Before
	```sh
	linux /boot/vmlinuz-6.14.0-15-generic root=UUID=e98e3d8d-a0b5-4d9a-80db-da976477c490 ro  quiet splash modprobe.blacklist=nouveau,nvidia,nvidia_drm,nvidia_uvm,nvidia_modeset,nvidia_fb use_vfio crashkernel=2G-4G:320M,4G-32G:512M,32G-64G:1024M,64G-128G:2048M,128G-:4096M $vt_handoff
	initrd	/boot/initrd.img-6.14.0-15-generic
	```
	After:
	```sh
	linux /boot/vmlinuz root=UUID=e98e3d8d-a0b5-4d9a-80db-da976477c490 ro  quiet splash modprobe.blacklist=nouveau,nvidia,nvidia_drm,nvidia_uvm,nvidia_modeset,nvidia_fb use_vfio crashkernel=2G-4G:320M,4G-32G:512M,32G-64G:1024M,64G-128G:2048M,128G-:4096M $vt_handoff
	initrd	/boot/initrd.img
	```

#### 40_custom file after edit:
```sh
#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.
menuentry 'Ubuntu (NVIDIA KVM passthrough)' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-e98e3d8d-a0b5-4d9a-80db-da976477c490' {
	recordfail
	load_video
	gfxmode $linux_gfx_mode
	insmod gzio
	if [ x$grub_platform = xxen ]; then insmod xzio; insmod lzopio; fi
	insmod part_gpt
	insmod ext2
	set root='hd1,gpt2'
	if [ x$feature_platform_search_hint = xy ]; then
	  search --no-floppy --fs-uuid --set=root --hint-bios=hd0,gpt2 --hint-efi=hd0,gpt2 --hint-baremetal=ahci0,gpt2 --hint='hd1,gpt2'  e98e3d8d-a0b5-4d9a-80db-da976477c490
	else
	  search --no-floppy --fs-uuid --set=root e98e3d8d-a0b5-4d9a-80db-da976477c490
	fi
	linux	/boot/vmlinuz root=UUID=e98e3d8d-a0b5-4d9a-80db-da976477c490 ro  quiet splash modprobe.blacklist=nouveau,nvidia,nvidia_drm,nvidia_uvm,nvidia_modeset,nvidia_fb use_vfio crashkernel=2G-4G:320M,4G-32G:512M,32G-64G:1024M,64G-128G:2048M,128G-:4096M $vt_handoff
	initrd	/boot/initrd.img

```

> ⚠️ **Note on Kernel Updates**  
> Ignore this note if step 2 part 4. Bonus is done successfully.  
> If your system updates to a new kernel, repeat **Step 1 to Step 3**.  
> Kernel updates change the file names (`vmlinuz-<version>` and `initrd.img-<version>`), so your custom GRUB entry must be updated accordingly.


### Step 3: Update grub
- On Ubuntu/Debian: `sudo update-grub`
- Other: `sudo grub-mkconfig -o /boot/grub/grub.cfg`

### Step 4: Create shell script to load VFIO

- Create new shell script:
`sudo nano /etc/init.d/use-vfio.sh`
- Paste this content to that file:
	```sh
	#!/bin/bash

	if grep -q "use_vfio" /proc/cmdline; then
		/usr/sbin/modprobe vfio
		/usr/sbin/modprobe vfio_iommu_type1
		/usr/sbin/modprobe vfio_pci
		/usr/sbin/modprobe vfio_virqfd
	fi
	```
- Save and exit nano.
- Make it executable: `chmod +x /etc/init.d/use-vfio.sh`

### Step 5: Create systemd service to run VFIO script at boot
- Create unit file:
`sudo nano /etc/systemd/system/use-vfio.service`
- Paste this content then save:
	```ini
	[Unit]
	Description=Load VFIO modules at boot for custom GRUB entry
	DefaultDependencies=no
	Before=sysinit.target

	[Service]
	Type=oneshot
	ExecStart=/etc/init.d/use-vfio.sh

	[Install]
	WantedBy=sysinit.target
	```
- Enable the service:
	`sudo systemctl enable use-vfio.service`
