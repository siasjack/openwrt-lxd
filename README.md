These scripts create LXD images for OpenWRT.
They download generic-rootfs.tar.gz from openwrt.org, modify it slightly, and add LXD metadata.

这个脚本是用来创建openwrt的lxd的镜像  
它会从openwrt.org上下载generic-rootfs.tar.gz文件系统压缩包，然后解压并修改他，然后增加lxd需要的metadata文件  

The resulting images have a couple of problems:    
- In order to complete booting, you need to run a script after starting the container, which creates missing devices in /dev
- The image should be launched in privileged mode, otherwise it can't create the missing devices.
- interactive ssh to the container does not work.  But non-interactive ssh does.  

当前的镜像存在以下几个问题：  
- 为了能完成启动，需要启动这个容器后运行一个脚本，用来创建一些/dev/下的字符型设备
- 镜像应该运行在特权模式下，否则无法完成上一条创建/dev下的设备
- 交互式的ssh暂时不支持，但是非交互式的ssh可以支持。我们可以用ash来实现交互式运行openwrt的命令

For a more complete image that does not have these problems, see https://github.com/mikma/lxd-openwrt

这有一个完整的镜像没有上面几个问题的，看这个：https://github.com/mikma/lxd-openwrt  

This may still be useful for platforms other than x86_64 or for quick testing.  
但是这个脚本对于初学者依然有用  

Naturally, you can (and should) run this script inside an LXD container.

运行下面的命令开始创建容器  
Run as follows:

	sudo ./build.sh {version}

Run without arguments to find out the available versions:

	./build.sh

openwrt的BB版本暂时不支持  
barrier_breaker is not usable.

The dhcp versions modify the network configuration so that the container gets its ip address from dhcp.  They also remove the wan interface.

If you use the original network configuration, which includes a DHCP server, other containers may start getting their ip addresses from the OpenWRT container (which may be useful, if you disable the LXD DHCP server).

rootfs tarballs are downloaded in ./cache, if they are not already there.
If you want to get fresh ones, delete the old ones first.

build.sh完成后，应生成import.sh和launch.sh两个脚本，分别用于导入镜像和启动镜像  

The resulting images are generated in ./target/{version}/image, which should be copied to the host.
There are also a couple of generated scripts:
import.sh imports the image
launch.sh <name> launches a container from the image

The openwrt container should be privileged:

	lxc launch {openwrt-image-alias} -c security.privileged=true {name}

The resulting container boots partially.  It becomes usable after running the script /root/init.sh.  You can exec init.sh using lxd exec, either directly, or through an interactive shell:  

使用如下命令运行init.sh来创建必须的几个/dev/下字符设备，必须使用特权模式（root用户运行）    

	lxc exec {container} /root/init.sh

or:  
运行如下命令可以进入镜像内的交互式命令行，等于登录了ssh  

	lxc exec {container} ash
	sh init.sh



init.sh uses mknod to create a few missing devices in /dev.  The container should be privileged in order to be able to run mknod.

我暂时还没有实现让init.sh随容易一起启动，这样就不需要上面的exec命令了  
I haven't been able to make init.sh run automatically from inside the container.  

运行“halt”命令来停止容器  lxd的命令无法停止此容器，lxc-stop也不可以停止  
To stop the container, run "halt" in it.  It does not seem to stop from the LXD tools.

If you try to run "halt" or "reboot" before completing the boot process, it won't work.  In order to get rid of such an unusable container, delete its rootfs from the host (/var/lib/lxd/containers/{container}/rootfs in pre-snap LXD).  You will then be able to delete the container after a host reboot.

The resulting container seems functional.
You can access the Luci Web Interface.
If you try to login to it with ssh, you get an error:  
如果你想用ssh root@openwrt-ip来登录此镜像的ssh服务器，会出现以下错误：  

	PTY allocation request failed on channel 0
	shell request failed on channel 0

But you can use scp, rsync, and run non-interactive commands with ssh.   
但是你可以使用如下几个命令scp，rsync，也可以运行一些非交互式的命令
barrier_breaker does not boot properly into multiuser mode.  However, you can halt or reboot it.  init.sh does not fix it.  This script does not modify rootfs in this case.  It just adds metadata.
