# helper ipmitool -I lan -U root -P ADMIN -H 192.168.1.xx chassis bootdev bios or pxe 

Quick rhel/centos unattendted installtion

# do not forget to copy iso in httpd file directory as per httpd.conf
# copy syslinux file to you know it.

# pxeboot
httpd-pxe.conf
----------
Alias /centos7_x64  /tftpboot/centos7_x64
<Directory /tftpboot/centos7_x64>
    Options Indexes FollowSymLinks
    # IP address you allow to access
    Require ip 127.0.0.1 192.168.1.0/8
</Directory>

Alias /ks /tftpboot
<Directory /tftpboot>
    Options Indexes FollowSymLinks
    # IP address you allow to access
    Require ip 127.0.0.1 192.168.1.0/8
</Directory>


dhcpd.conf
----------
cat /etc/dhcp/dhcpd.conf
# dhcpd.conf
omapi-port 7911;

default-lease-time 43200;
max-lease-time 86400;



ddns-update-style none;

option domain-name "whatisyourdomain.no";
option domain-name-servers 192.168.1.00;
option ntp-servers none;

allow booting;
allow bootp;

option fqdn.no-client-update    on;  # set the "O" and "S" flag bits
option fqdn.rcode2            255;
option pxegrub code 150 = text ;




# Bootfile Handoff
next-server 192.168.1.37;
option architecture code 93 = unsigned integer 16 ;
if option architecture = 00:06 {
  filename "grub2/shim.efi";
} elsif option architecture = 00:07 {
  filename "grub2/shim.efi";
} elsif option architecture = 00:09 {
  filename "grub2/shim.efi";
} else {
  filename "pxelinux.0";
}

log-facility local7;

include "/etc/dhcp/dhcpd.hosts";
# xxx.no
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.1 192.168.1.200;
  option subnet-mask 255.255.255.0;
  option routers 192.168.1.1;
}

subnet 10.0.1.0 netmask 255.255.255.0 {
  range 10.0.1.1 10.0.1.200;
  option subnet-mask 255.255.255.0;
  option routers 10.0.1.1;
}

#pxe-default
install
firewall --disabled
url --url="http://192.168.1.37/centos7_x64"
# Network information
network  --bootproto=dhcp --device=em1 --ipv6=auto --activate
network  --hostname=cloud01-node01.hostname.no
# Root password [i used here 000000]
rootpw --iscrypted $1$xYUugTf4$4aDhjs0XfqZ3xUqAg7fH3.
# System authorization information
auth  useshadow  passalgo=sha512
# Use graphical install
graphical
firstboot --enable
keyboard --vckeymap=no --xlayouts='no','us'
# System language
lang en_US.UTF-8
# SELinux configuration
selinux disabled
eula --agreed
reboot
ignoredisk --only-use=sda
# Installation logging level
logging level=info
# System timezone
timezone Europe/Amsterdam
# System services
services --enabled="chronyd"
# System bootloader configuration
bootloader location=mbr
autopart --type=lvm
# Clear the Master Boot Record
zerombr
# Partition clearing information
clearpart --drives=sda --all --initlabel
%packages --ignoremissing
@core
%end
%post
mkdir -p /root/.ssh
chmod 700 /root/.ssh
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABZx8bi8pdCX1G3KoOp+/sSJ notroot@yourdomain.no" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
restorecon -R /root/.ssh/
%end

#pxe-menu default
default menu.c32
prompt 0
timeout 3000
ONTIMEOUT LOCAL

MENU TITLE MyCo  PXE Menu

LABEL LOCAL
MENU LABEL Boot from local drive
COM32 chain.c32
APPEND hd0 0

LABEL CentOS7_X64
MENU LABEL Install CentOS 7 X64 minimal
KERNEL centos7_x64/images/pxeboot/vmlinuz
append vga=normal initrd=centos7_x64/images/pxeboot/initrd.img locale=en_US.UTF-8 ramdisk_size=1 inst.repo=http://192.168.1.xx/centos7_x64 inst.stage2=http://192.168.1.xx/centos7_x64 inst.ksdevice=em1 inst.ks=http://192.168.1.xx/ks/ks.cfg inst.method=http://192.168.1.xx/centos7_x64

Install and start services xinetd, httpd, dhcpd. Make shoure tftpboot file points to your tftp directory

echo "######Starting PXE Boot setup#####"
systemctl restart xinetd
systemctl restart httpd
systemctl restart dhcpd
echo "#######Done######"
