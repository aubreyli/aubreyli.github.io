---
layout: post
#comments: true
categories: linux
---

## Ubuntu console login
On ubuntu 18.04, to disable GUI on boot, run:

	sudo systemctl set-default multi-user.target

To enable GUI again issue the command:

	sudo systemctl set-default graphical.target

---

## Fix up GRUB install
Press C for the Grub command line, which will drop you back to a prompt:
Now, enter these commands in sequence:

	set root=(hd0,gpt2)
	linux /casper/vmlinuz.efi root=/dev/sda2
	initrd /casper/initrd.lz
	boot

After system boot, install / update some packages related to the 32/64-bit UEFI:

	sudo apt-get install grub-efi-ia32-bin

Then, tell grub to finish setting itself up properly against the drive on your PC where Ubuntu is installed. Remember earlier when we noted both the Disk and Partitioninfo we used the partition, now we want the Disk:

	sudo grub-install /dev/<yourDisk>

	sudo grub-update

## Add an account
Create a new user with home folder and bash

	sudo useradd --create-home --user-group --shell /bin/bash <username>
