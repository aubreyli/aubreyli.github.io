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

## Change hostname
	sudo hostnamectl set-hostname alc-machine

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
Create a new user with home folder and bash, and grant sudoer

	sudo useradd --create-home --user-group --shell /bin/bash aubrey
	sudo usermod -aG sudo aubrey

## Cron tips
Sync up timezone with localtime and restart cron service

	ps -ef | grep cron
	cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
	systemctl restart cron

## build linux kernel
	apt-get update
	apt-get install kernel-package libncurses5-dev fakeroot wget bzip2 build-essential
	make-kpkg clean
	fakeroot make-kpkg --jobs `nproc` --initrd kernel_image
	or
	fakeroot make-kpkg --jobs `nproc` --initrd --append-to-version=-custom kernel_image kernel_headers
	sudo make deb-pkg LOCALVERSION=-custom KDEB_PKGVERSION=$(make kernelversion)-1
	make -j `getconf _NPROCESSORS_ONLN` 
	make deb-pkg LOCALVERSION=-custom

## submit LKP job
	aubrey@inn:~$ lkp queue -b aubrey-fast_idle/4.8.x -k 59d15ffe62d16350cf800f10a4d4a86cd3cd7b71 -t lkp-bdw-ep2 netperf.yaml -r 3 -q vip -e aubrey.li@intel.com
	aubrey@inn:~$ lkp wtmp lkp-bdw-ep2 
	aubrey@inn:~$ kbuild-queue -b aubrey-fast_idle/avx_v9 -k d52843ee72fe3e7d2114c1c83b84d90ada6c2591 -w -c config
	$lkp ncompare -k -a d34a27ce2f54bfa985373d40efb28e48c8a07a3f 6d616e5e762c00b9d7f4b160028922bcf12236cc
	$lkp compare -at d34a27ce2f54bfa985373d40efb28e48c8a07a3f 6d616e5e762c00b9d7f4b160028922bcf12236cc -g schbench -g performance-100%-ucode=0x200004d
	$lkp power_reboot lkp-skl-2sp5
	$lkp serial lkp-skl-2sp5
	$ssh lkp-skl-2sp5, root, asdf
	list queued job: $ lkp lq lkp-skl-2sp5
	list tbox:
	$ ls /lkp/wtmp/lkp-*
	$ ls /lkp/lkp/src/hosts
	lkp queue -t lkp-cfl-d1 -b internal-aubrey-fast_idle/idle-cpumask-v5 -q vip borrow-1d-1d.yaml
- check job queue:
	ls /lkp/jobs/queued/int/lkp-cfl-d1
	ls /lkp/jobs/queued/vip/lkp-cfl-e1/
- my repo:
	ssh bee; cd /git/aubrey/fast_idle.git;
	ssh inn; cd /c/repo/linux; git branch -a | grep aubrey

## netperf
	netserver
	netperf -H 127.0.0.1 -l 60 -t TCP_RR

## grub2 default entry
	GRUB_DEFAULT="1>3"
	sudo update-grub

## kprobe
	1) build kernel with CONFIG_KPROBE_EVENT=y
	2) echo 'p:aubrey_probe intel_idle dev=%di drv=%si index=%dx' > /sys/kernel/debug/tracing/kprobe_events
	3) echo 'r:aubrey_probe intel_idle $retval' > /sys/kernel/debug/tracing/kprobe_events
	4) trace-cmd record -T -e aubrey_probe
	5) trace-cmd report

## process affinity
	1) set affinity when the process is runing:
   	taskset -cp 0-21 `pgrep netserver`
	2) set affinity when the process is invoked:
   	taskset -c 22-43 netserver
	3) check which CPU the process is assigned to:
   	ps -mo pid,tid,%cpu,psr -p `pgrep netserver`

## unzip xz
	xz -d dmesg.xz

## perf with intel_pt
	1) perf-with-kcore record pt_tmp -C0 -e intel_pt/cyc/k --filter="filter tick_nohz_idle_enter / tick_nohz_idle_exit" -- sleep 2
	2) perf-with-kcore script pt_tmp --itrace=cre --ns -Fcomm,pid,tid,cpu,time,ip,sym,symoff,dso,addr,flags,callindent

## submit patch
	1) git format-patch -2 -v1 --subject-prefix="RFC PATCH" --cover-letter
	2) git send-email --to tglx@linutronix.de --to peterz@infradead.org --to len.brown@intel.com --to rjw@rjwysocki.net --to ak@linux.intel.com --to tim.c.chen@linux.intel.com --to arjan@linux.intel.com --to paulmck@linux.vnet.ibm.com --to yang.zhang.wz@gmail.com --cc x86@kernel.org --cc linux-kernel@vger.kernel.org *.patch

## tmux tips
	C-b "	上下分屏
	C-b %	左右分屏
	C-b d   detach
	tmux attach-session -t my_session

## setup simics networking
	1) sudo apt-get install uml-utilities bridge-utils
	2) sudo tunctl -u `whoami` -t sim_tap0
	3) sudo ifconfig sim_tap0 192.168.1.10
	4) simics console running> connect-real-network-host interface = sim_tap0

## Linux kernel contributor list
	1) git log v3.0..v4.15 | grep Author | cut -d ":" -f 2 | sort | uniq -c | sort -nr | head -n 500
	2) git log --graph --abbrev-commit --decorate --format=format:'%C(bold blue)%h%C(reset) - %C(bold green)(%ar)%C(reset) %C(white)%s%C(reset) %C(dim white)- %an%C(reset)%C(auto)%d%C(reset)' --all

## ubuntu vnc
	1) ubuntu-gnome-desktop
	2) gnome-applets

## git lkp
	1) $ git fetch ssh://aubrey-skl/home/aubrey/work/linux-tip core_scheduling:core_scheduling
	2) $ git checkout core_scheduling
	3) $ git push -u origin core_scheduling
	4) $ git push origin :branch-name (delete remote branch)

## perf profiling
	1) perf record -a -g -- sleep 10
	2) perf record --all-kernel
	3) perf report -g --no-children

## ssh via socks proxy
	ssh -o ProxyCommand='nc -x proxy-prc.intel.com:1080 %h %p' aubrey@45.32.228.76
	For dnf-system: ssh -o ProxyCommand="connect-proxy -S child-prc.intel.com:1080 %h %p" root@8.140.143.216

## perf package
	apt install libdw-dev libgtk2.0-dev libslang2-dev systemtap-sdt-dev libunwind-dev libperl-dev python-dev libnuma-dev libbabeltrace-dev binutils-dev libiberty-dev libaudit-dev default-jre openjdk-11-jdk asciidoc

## cpu online/offline
	echo 0 | sudo tee /sys/devices/system/cpu/cpu53/online
	echo off | sudo tee /sys/devices/system/cpu/smt/control

## git intel-next
	git push origin HEAD
	git push origin HEAD --force
	git log origin/branch_name
	git fetch ../linux-stable refs/tags/v5.10.60:refs/tags/v5.10.60 --no-tags
	git push origin v5.10.60

## perf branch profiling
	sudo perf record -b -a -g sleep 2
	sudo perf report --samples 10

## batch add acked-by for anolis

	If you need to add Acked-by lines to, say, the last 10 commits (none of which is a merge), use this command:

	git filter-branch --msg-filter '
		cat &&
		echo "Reviewed-by: Artie Ding <artie.ding@linux.alibaba.com>"
	' HEAD~10..HEAD

## pull request
	https://gitee.com/anolis/cloud-kernel/pull/new/anolis:intel-siov-ANBZ...anolis:devel-5.10?source_repo=20938569

## git commit and author date and merged tag
	git formatpatch -1 --pretty=fuller
	git tag --contains 9f8c7baedabc9693fbd7890f8fda40578bde4f73

## check network port
	netstat -tpln | grep 590

## check disk rd/wr speed
	1) write:
   	time dd if=/dev/zero of=/tmp/test bs=8k count=100000
	2) read:
  	 time dd if=/tmp/test of=/dev/null bs=8k
	3) read/write:
   	time dd if=/tmp/test of=/var/test bs=64k

## cross compile
	cd $KERNEL_SOURCE
	make ARCH=$KERNEL_ARCH O=$KERNEL_BUILD $KERNEL_CONF

	cd $KERNEL_BUILD
	make ARCH=$KERNEL_ARCH CROSS_COMPILE=$TARGET- V=1 vmlinux modules

	make -j120 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-

## find file containing string and move it
	find . -type f -exec grep -lr "Intel-SIG" {} \; -exec mv {} ./tmp/ \;

## turbostat usage
	sudo turbostat -c 56 --show Package,Core,CPU,Avg_MHz,Busy%,Bzy_MHz,TSC_MHz,CoreTmp,PkgTmp

## show topology
	lstopo --of ascii | less

## grubby usage
	(1) grubby --info=ALL | grep -E "^kernel|^index"
	(2) grubby --set-default="/boot/vmlinuz-4.18.0-193.1.2.el8_2.x86_64"
	(3) grubby --set-default-index=2
	(4) grubby --default-kernel
	(5) grubby --default-index
	(6) grubby --update-kernel=/boot/vmlinuz-$(uname -r) --args="ipv6.disable=1"
	(7) grubby --update-kernel=ALL --args="ipv6.disable=1"
	(8) grubby --update-kernel=/boot/vmlinuz-$(uname -r) --remove-args="ipv6.disable=1"
	(9) grubby --update-kernel=ALL --remove-args="ipv6.disable=1"
	(10) ls -l /boot/loader/entries/*
	(11) grubby --remove-kernel=menu_index
	(12) grubby --remove-kernel=old_kernel

## set the date
	sudo hwclock --set --date="2023-04-26 10:30:00"
	sudo date --set="2023-04-26 10:30:00"

## fix qemu GTK initialization issue
	xhost si:localuser:root

## scp thru a jumper
	scp -J root@10.238.153.101 root@192.168.135.28:~/file.txt .

## GRUB2 enable debug message
	set debug=all
	grub-install --debug-image=all

## Disable sleep/auto-suspend
	sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target

## Download unsloth model (hf-mirror or modelscpoe)
	git clone --no-checkout https://hf-mirror.com/unsloth/DeepSeek-R1-GGUF 
  	GIT_LFS_SKIP_SMUDGE=1 git clone https://www.modelscope.cn/unsloth/DeepSeek-R1-0528-GGUF.git
	cd DeepSeek-R1-GGUF/
	# 建议nohup执行, 预计至少需要半天时间, 同时确保磁盘容量足够400G.
 	git lfs pull --include="Q4_K_M/*.gguf"
