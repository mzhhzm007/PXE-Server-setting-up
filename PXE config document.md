# PXE服务器搭建
Requirement：The automated testing environment needs a stable linux system, so it is necessary to set up a server to provide a stable linux system<br>

This instruction introduced how to set up a PXE Server which install a system through network<br>

The working principle can refer to the Internet.

which is required
* DHCP
* tftp-server
* httpd
* syslinux
* system files

Main Step
1. install dhcp<br>
```
    yum -y install dhcp
```
then modify the dhcp config file
```
vim /etc/dhcp/dhcpd.conf 
```
```
#
# DHCP Server Configuration file.
#   see /usr/share/doc/dhcp*/dhcpd.conf.sample
#   see 'man 5 dhcpd.conf'
#
log-facility local7;
default-lease-time 300;
max-lease-time 7200;
allow booting;
allow bootp;
option option-128 code 128 = string;
option option-129 code 129 = text;

subnet 192.168.2.2 netmask 255.255.255.0 {
	range 192.168.2.10 192.168.2.20; # dhcp ip address pool
	option routers 192.168.2.1; # route
	next-server 192.168.2.2; # ip address of tftp
	filename "pxelinux.0"; # PXE file
}
```
2. intall tftp-server<br>
```
    yum install tftp-server
```
then modify the tftp config file
```
vim /etc/xinetd.d/tftp
```
main config as below

```
# default: off
# description: The tftp server serves files using the trivial file transfer \
#	protocol.  The tftp protocol is often used to boot diskless \
#	workstations, download configuration files to network-aware printers, \
#	and to start the installation process for some operating systems.
service tftp
{
	disable	= no            # make sure it is "no"      
	socket_type		= dgram
	protocol		= udp
	wait			= yes
	user			= root
	server			= /usr/sbin/in.tftpd
	server_args		= -s /home/ATC/tftpboot  # tftp directory
	per_source		= 11
	cps			= 100 2
	flags			= IPv4
}
```


3. install httpd
```
yum install httpd -y
```
use the default config, so the httpd directory should be <font color=#FF7256>`/var/www/html`</font><br>
also the directory can be changed in `/etc/httpd/conf/httpd.conf`

4. prepare a whole linux system files and put them in httpd directory (mine is `/var/www/html`)<br>
you can mount an Linux ISO file and then copy all the files to httpd directory<br>
my system is `CentOS 6.5 x86_64`
so my dir is as below

    ![image](/files/pic_httpddir.png)<br>

5. prepare necessary linux files in tftp dir<br>
my tftp dir is `/home/ATC/tftpboot` so my operation is as below<br>
(my CentOS files are already in `/var/www/html`)
```
cp /var/www/html/CentOS-6.5/os/x86_64/images/pxeboot/{vmlinuz,initrd.img} /home/ATC/tftpboot/ 
cp /var/www/html/CentOS-6.5/os/x86_64/isolinux/boot.msg /home/ATC/tftpboot/
cp /var/www/html/CentOS-6.5/os/x86_64/isolinux/splash.jpg /home/ATC/tftpboot/ 
cp /var/www/html/CentOS-6.5/os/x86_64/isolinux/vesamenu.c32 /home/ATC/tftpboot/
```

6. we need a file called "pxelinux.0", it is provided by syslinux<br>
`yum -y install syslinux`<br>
copy pxelinux.0 to tftp dir<br>
`cp /usr/share/syslinux/pxelinux.0 /home/ATC/tftpboot`<br> 
the pxelinux.0 works rely on isolinux.cfg, we copy it to tftp directory and rename it to "default"
    ```
    mkdir /home/ATC/tftpboot/pxelinux.cfg
    cd /home/ATC/tftpboot/pxelinux.cfg
    cp /var/www/html/CentOS-6.5/os/x86_64/isolinux.cfg /home/ATC/tftpboot/pxelinux.cfg/default
    ```
    the file "default" shows the information when install system, we should modify the file to adapt our system source
    my example is as below<br>
`vim default`

    ```
    default vesamenu-6.5-64.c32 ##################modified##################
    #prompt 1
    timeout 600
    
    display boot-6.5-64.msg ##################modified##################
    
    menu background splash-6.5-64.jpg ##################modified##################
    menu title Welcome to CentOS 6.5!
    menu color border 0 #ffffffff #00000000
    menu color sel 7 #ffffffff #ff000000
    menu color title 0 #ffffffff #00000000
    menu color tabmsg 0 #ffffffff #00000000
    menu color unsel 0 #ffffffff #00000000
    menu color hotsel 0 #ff000000 #ffffffff
    menu color hotkey 7 #ffffffff #ff000000
    menu color scrollbar 0 #ffffffff #00000000
    
    label linux
      menu label ^Install or upgrade an existing system
      menu default
      kernel vmlinuz-6.5-64  ##################modified##################
      append initrd=initrd-6.5-64.img xdriver=vesa method=http://192.168.2.2/CentOS-6.5/os/x86_64 ##################modified##################
    label vesa
      menu label Install system with ^basic video driver
      kernel vmlinuz
      append initrd=initrd.img xdriver=vesa nomodeset
    label rescue
      menu label ^Rescue installed system
      kernel vmlinuz
      append initrd=initrd.img rescue
    label local
      menu label Boot from ^local drive
      localboot 0xffff
    label memtest86
      menu label ^Memory test
      kernel memtest
      append -
    ```
the content in "default" file which line marked by `#####modified####` should be consistent with your files in tftp directory<br>
my file is `vesamenu-6.5-64.c32` `boot-6.5-64.msg` `vmlinuz-6.5-64` `initrd-6.5-64.img`<br>
finally my tftp dir is as below:<br>
(you can rename your files like my example)<br>

  ![image](/files/pic_tftpdir.png)<br>


7. change SELINUX to disabled
```
vim /etc/selinux/config

change SELINUX=enforcing to SELINUX=disabled
```
reboot and check: `getenforce` should be disabled


8. make everythins ready<br>
* `service iptables stop`<br>
make the iptables is off
* `service dhcpd start`<br>
test the dhcp is running, you can test it by use another pc
* `service xinetd start`<br>
test the tftp service, you can `tftp {your tftp server ip}` and get one file to ensure it is running normally
* `service httpd start`<br>
test the httpd service, you can `curl http://{your httpd server ip}/CentOS-6.5/os/x86_64` to check whether you can get it

finally, you can start to test install a system by PXE.

   
   
   

   
