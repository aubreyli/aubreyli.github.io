---
layout: post
#comments: true
categories: linux
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

### Connect serial console

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

---

## Clone the virtual machine

It's easy to create another VM to copy from current VM with a command below:

	root@aubrey-skl:/home/aubrey# virt-clone --original aubrey-kvm1 --name aubrey-kvm2 --file ubuntu1804_2.img

### Change MAC address

The cloned machine has the same mac address, which needs to be modified

	root@aubrey-skl:/home/aubrey# virsh dumpxml aubrey-kvm2 | grep 'mac address'
	<mac address='52:54:00:95:32:69'/>
	root@aubrey-skl:/home/aubrey# vi /etc/libvirt/qemu/aubrey-kvm2.xml

### Change IP address
Then edit the network, to change the IP address

	virsh  net-list
	virsh  net-edit  $NETWORK_NAME    # mine is "default"

Find the <dhcp> section, restrict the dynamic range and add host entries for your VMs

	<ip address='192.168.122.1' netmask='255.255.255.0'>
		<dhcp>
			<range start='192.168.122.2' end='192.168.122.254'/>
			<host mac='52:54:00:a4:1d:df' name='alkvm-ubuntu1' ip='192.168.122.28'/>
			<host mac='52:54:00:95:32:69' name='alkvm-ubuntu2' ip='192.168.122.38'/>
		</dhcp>
	</ip>

Then, reboot the VM (or restart its DHCP client, e.g. ifdown eth0; ifup eth0)

That probably does not work, if so try this:

	virsh  net-destroy  $NETWORK_NAME  
	virsh  net-start    $NETWORK_NAME  

and restart the VM's DHCP client.

If that still doesn't work,  we might have to
- stop the libvirtd service
- kill any dnsmasq processes that are still alive
- start the libvirtd service
