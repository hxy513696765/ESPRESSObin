# ESPRESSObin

## 编译内核

### 下载编译器

[https://releases.linaro.org/components/toolchain/binaries/5.2-2015.11-2/aarch64-linux-gnu/gcc-linaro-5.2-2015.11-2-x86_64_aarch64-linux-gnu.tar.xz](https://releases.linaro.org/components/toolchain/binaries/5.2-2015.11-2/aarch64-linux-gnu/gcc-linaro-5.2-2015.11-2-x86_64_aarch64-linux-gnu.tar.xz)

### 下载内核代码

	git clone https://github.com/MarvellEmbeddedProcessors/linux-marvell
	git checkout linux-4.4.8-armada-17.02-espressobin

### 配置和编译内核(默认配置)

	make O=out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- mvebu_v8_lsp_defconfig
	make O=out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j8

### 编译内核支持ubuntu route

使用下面内核配置

[ubuntu kenrel config](./ubuntu_config)

基本设置

网络配置使能联网下载软件包

	ifconfig eth0 up
	dhclient wan

安装相关的软件包

	apt-get update
	apt-get install bridge-utils samba dnsmasq-base iptables


开启路由(操作完成后就是一个gateway的角色)

	brctl addbr br0
	ifconfig eth0 0.0.0.0 up
	ifconfig wan 0.0.0.0 up
	ifconfig lan0 0.0.0.0 up
	ifconfig lan1 0.0.0.0 up
	brctl addif br0 lan0
	brctl addif br0 lan1
	ifconfig br0 192.168.22.1

	/etc/init.d/smbd stop
	/etc/init.d/smbd start

	dnsmasq --interface=br0 --dhcp-range=br0,192.168.22.2,192.168.22.199,12h
	echo 1 > /proc/sys/net/ipv4/ip_forward
	iptables -t nat -A POSTROUTING -o wan -j MASQUERADE
	dhclient wan

测试是否设置成功ping google

	ping 8.8.8.8

测试是否设置成功ping local machine

	ping 192.168.1.100

## SD卡启动系统

查看SD卡状态(下面是笔者主机信息)

	lsblk
	NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
	sda      8:0    0 931.5G  0 disk
	├─sda1   8:1    0    50G  0 part
	├─sda2   8:2    0     1K  0 part
	├─sda5   8:5    0   673G  0 part /home
	├─sda6   8:6    0 517.7M  0 part /boot
	├─sda7   8:7    0   200G  0 part /
	└─sda8   8:8    0     8G  0 part [SWAP]
	sdb      8:16   1  14.9G  0 disk
	└─sdb1   8:17   1  14.9G  0 part

清除SD卡(这里假设是/dev/sdb)内所有数据,可以用lsblk查看SD卡状态

	sudo dd if=/dev/zero of=/dev/sdb bs=1M count=100

在SD上创建一个分区(sdb1)

	(echo n; echo p; echo 1; echo ''; echo ''; echo w) | sudo fdisk /dev/sdb

格式化分区为EXT4格式

	sudo mkfs.ext4 /dev/sdb1

把SD卡挂在到开发主机的/mnt/sdcard目录下

	sudo mkdir -p /mnt/sdcard
	sudo mount /dev/sdb1 /mnt/sdcard

将根文件系统(buildroot, ubuntu, yocto)解压到/mnt/sdcard/下(这里统一用rootfs.tar.gz表示)

	sudo tar -xvf rootfs.tar.gz -C /mnt/sdcard/

将内核和DTB拷贝到根文件系统的boot目录下

	sudo mkdir -p /mnt/sdcard/boot
	sudo cp Image /mnt/sdcard/boot/
	sudo cp marvell/armada-3720-community.dtb /mnt/sdcard/boot/

卸载SD卡

	sudo umount /mnt/sdcard

设置Uboot环境变量

	setenv image_name boot/Image
	setenv fdt_name boot/armada-3720-community.dtb
	setenv bootmmc 'mmc dev 0; ext4load mmc 0:1 $kernel_addr $image_name;ext4load mmc 0:1 $fdt_addr $fdt_name;setenv bootargs $console root=/dev/mmcblk0p1 rw rootwait; booti $kernel_addr - $fdt_addr'
	save

使用bootmmc启动开发板

	run bootmmc

## Ubuntu 14.04根文件系统制作

[ubuntu-base-14.04-core-arm64.tar.gz下载地址](http://cdimage.ubuntu.com/ubuntu-base/releases/14.04/release/)

将ubuntu core解压到本地

	mkdir ubunturootfs
	sudo tar xzvf ubuntu-base-14.04-core-arm64.tar.gz -C ubunturootfs/

修改默认启动级别(etc/init/rc-sysinit.conf)

	DEFAULT_RUNLEVEL=3

去除root用户登录密码(etc/passwd)

	root::0:0:root:/root:/bin/bash

创建一个串口初始化配置文件(etc/init/ttyMV0.conf)

	start on stopped rc or RUNLEVEL=[12345]
	stop on runlevel [!12345]
	respawn
	exec /sbin/getty -L 115200 ttyMV0 vt100 -a root

将内核和DTB拷贝到根文件系统的boot目录下(制作好的文件系统就可以直接拷贝到SD卡中使用)

	cp Image boot/
	cp armada-3720-community.dtb boot/

## NFS服务器配置

安装必要的软件包

	apt-get install nfs-kernel-server

设置需要export的目录

	mkdir -p /espressobin_export/music

设置相关权限

	chmod -R 777 espressobin_export/
	chown -R nobody:nogroup espressobin_export/

绑定目录到NFS的路径

	mount --bind /home/music /espressobin_export/music

如果需要开机就bind的话在fstab里添加如下内容

	/home/music   /espressobin_export/music   none   bind   0   0

确认NFS服务器和客户端都在相同的域(/etc/idmapd.conf)

	[Mapping]
	Nobody-User = nobody
	Nobody-Group = nogroup

添加如下内容到/etc/exports

	/espressobin_export       192.168.22.0/24(rw,fsid=0,no_subtree_check,sync)
	/espressobin_export/music 192.168.22.0/24(rw,nohide,no_subtree_check,sync)

开启NFS

	exportfs -ra

每次有修改/etc/exports文件都要执行下面命令

	service nfs-kernel-server restart

在客户端操作如下(192.168.22.1是NFS服务器)

	sudo mount -o proto=tcp,port=2049 192.168.22.1:/espressobin_export /media

## 使用过程中遇到的问题和解决办法

Ubunt14.04网络配置

手动配置(开机不会在配置网络上卡吨)

	TBD

使用DHCP配置(添加下面内容到/etc/network/interfaces)

	auto wan
	iface wan inet dhcp

使用DHCP配置开机时配置始终不成功日志如下

	Waiting for network configuration...
	Waiting up to 60 more seconds for network configuration...
	Booting system without full network configuration...

进入系统后手动开启网卡,顺序不能颠倒,否则wan始终打不开

	ifconfig eth0 up
	ifconfig wan up

查看此时的路由表

	root@localhost:~# route
	Kernel IP routing table
	Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
	default         192.168.1.1     0.0.0.0         UG    0      0        0 wan
	192.168.1.0     *               255.255.255.0   U     0      0        0 wan
