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

---
## KVM create virtual machine
Install GuestOS and create Virtual Machine. Install GuestOS Ubuntu 18.04 on text mode via network, it's OK on Console or remote connection with Putty and so on.

	root@aubrey-skl:/home/aubrey# virt-install \
	--name aubrey-kvm1 \
	--ram 2048 \
	--disk path=/home/aubrey/kvm/ubuntu1804.img,size=15 \
	--vcpus 64 \
	--os-type linux \
	--os-variant ubuntu18.04 \
	--graphics none \
	--console pty,target_type=serial \
	--location=https://mirrors.aliyun.com/ubuntu/ubuntu/dists/bionic/main/installer-amd64/ \
	--extra-args 'console=ttyS0,115200n8 serial'

after finishing installation, back to KVM host and shutdown the guest like follows

	root@aubrey-skl:/home/aubrey# virsh shutdown aubrey-kvm1

mount guest's disk and enable a service like follows
	root@aubrey-skl:/home/aubrey# guestmount -d ubuntu1804 -i /mnt 
	root@aubrey-skl:/home/aubrey# ln -s /mnt/lib/systemd/system/getty@.service /mnt/etc/systemd/system/getty.target.wants/getty@ttyS0.service 
	root@aubrey-skl:/home/aubrey# umount /mnt

start guest again, if it's possible to connect to the guest's console, it's OK all
	root@aubrey-skl:/home/aubrey# virsh start aubrey-kvm1 --console 
	Ubuntu 18.04 LTS ubuntu ttyS0
	alkvm-ubuntu1 login:
