### SMT service

* Install and configure SMT service
 
  * configuration files:

```bash
/etc/sysconfig/smt-client
/etc/logrotate.d/smt-client
/etc/logrotate.d/smt
/etc/apache2/conf.d/smt_mod_perl.conf
/etc/apache2/conf.d/smt_support.conf
/etc/apache2/smt-mod_perl-startup.pl
/etc/smt.d/smt-cron.conf
/etc/smt.conf
```

### Setting up DNSMASQ

```bash
zypper in -y dnsmasq
```

- edit file _/etc/dnsmasq.conf_

```yaml
# Log lots of extra information about DHCP transactions.
log-dhcp

# Include another lot of configuration options.
conf-dir=/etc/dnsmasq.d/,*.conf

# log settings include /etc/logrotate.d/dnsmasq
log-facility=/var/log/dnsmasq.log

# Never forward addresses in the non-routed address spaces.domain-needed
bogus-priv

# Set the domain for dnsmasq. this is optional, but if it is set, it
# does the following things.
# 1) Allows DHCP hosts to have fully qualified domain names, as long
#     as the domain part matches this setting.
# 2) Sets the "domain" DHCP option thereby potentially setting the
#    domain of all systems configured by DHCP
# 3) Provides the domain part for "expand-hosts"
domain=sestest

# Set this (and domain: see below) if you want to have a domain
# automatically added to simple names in a hosts-file.
expand-hosts

# Add local-only domains here, queries in these domains are answered
# from /etc/hosts or DHCP only.
local=/sestest/

# If you want dnsmasq to listen for DHCP and DNS requests only on
# specified interfaces (and the loopback) give the name of the
# interface (eg eth0) here.
# Repeat the line for more than one interface.
interface=eth0
# Or you can specify which interface _not_ to listen on
except-interface=eth1
# Or which to listen on by address (remember to include 127.0.0.1 if
# you use this.)
listen-address=192.168.122.1
# If you want dnsmasq to provide only DNS service on an interface,
# configure it as shown above, and then use the following line to
# disable DHCP and TFTP on it.
no-dhcp-interface=eth1

# On systems which support it, dnsmasq binds the wildcard address,
# even when it is listening on only some interfaces. It then discards
# requests that it shouldn't reply to. This has the advantage of
# working even when interfaces come and go and change address. If you
# want dnsmasq to really bind only the interfaces it is listening on,
# uncomment this option. About the only time you may need this is when
# running another nameserver on the same machine.
bind-interfaces

# Uncomment this to enable the integrated DHCP server, you need
# to supply the range of addresses available for lease and optionally
# a lease time. If you have more than one network, you will need to
# repeat this for each network on which you want to supply DHCP
# service.
dhcp-range=192.168.122.10,192.168.122.20,48h
dhcp-range=lan,192.168.122.100, 192.168.122.200

# Override the default route supplied by dnsmasq, which assumes the
# router is the same machine as the one running dnsmasq.
dhcp-option=lan,3,192.168.122.1

# Add other name servers here, with domain specs if they are for
# non-public domains.
server=8.8.8.8

# If you don't want dnsmasq to read /etc/hosts, uncomment the
# following line.
no-hosts
# or if you want it to read another file, as well as /etc/hosts, use
# this.
addn-hosts=/etc/dnsmasq_static_hosts.conf

# set up router 
dhcp-option=option:router,192.168.122.1
# and ntp server for dhcp clients
dhcp-option=option:ntp-server,192.168.122.1
```

* next we need to configure DNS and use static IP for hosts that will be installed over PXE. 

- edit */etc/dnsmasq_static_hosts.conf* , example (same syntax as for /etc/hosts):

```bash
192.168.122.10 osd-node1.sestest osd-node1
192.168.122.11 osd-node2.sestest osd-node2
```

- edit */etc/dhcp_permanent_leases.conf*, example:

```bash
# dhcp lease based on MAC
dhcp-host=00:02:c9:53:b0:0c,osd-node1,192.168.122.10,infinite
dhcp-host=68:05:ca:2d:19:ac,osd-node2,192.168.122.11,infinite
```

- and we're going to use separate file for our pxe specific settings */etc/dnsmasq.d/pxe_settings.conf*

```yaml
# enable tftp
enable-tftp

# set up tftp root directory
tftp-root=/srv/tftpboot

# BIOS x86
# if client sents its architecture specification to be legacy x86_64
# dnsmasq will use tag x86PC
dhcp-match=x86PC, option:client-arch, 0
# if dnsmasq find out tag x86PC it will send this file to client to use 
# it for the installation
dhcp-boot=tag:x86PC,legacy/x86_64/pxelinux.0,192.168.122.1

# EFI x86_64
# if client sents its architecture specification to be UEFI x86_64
# dnsmasq will use tag BC_EFI
dhcp-match=BC_EFI, option:client-arch, 7
# if dnsmasq find out tag BC_EFI it will send this file to client to use 
# it for the installation
dhcp-boot=tag:BC_EFI,EFI/shim-sles.efi,192.168.122.1
```

* set up logrotate

  * create file */etc/logrotate.d/dnsmasq* and restart logrotate daemon

```yaml
/var/log/dnsmasq.log {
monthly
missingok
notifempty
delaycompress
sharedscripts
postrotate
  [ ! -f /var/run/dnsmasq.pid ] || kill -USR2 `cat /var/run/dnsmasq.pid`
endscript
create 0640 dnsmasq root
}
```

### Setting up PXE
 
* create directories:

```bash
mkdir -p /srv/install/x86_64/sles12sp3/cd1/
```

* mount ISO images (SLES, SES) to *mountpoint* and copy files to newly created directories. 

```bash
rsync -avP /SLES_mountpoint/ /srv/install/x86_64/sles12sp3/cd1/
```

* create structure in tftpboot:

```bash 
mkdir -p /srv/tftpboot/EFI
mkdir -p /srv/tftpboot/legacy/x86_64/pxelinux.cfg
```

* set up BIOS - legacy boot environment

```bash
cd /srv/install/x86_64/sles12sp3/cd1/boot/x86_64/loader
cp -a linux initrd message /srv/tftpboot/legacy/x86_64/
cp -a isolinux.cfg /srv/tftpboot/legacy/x86_64/pxelinux.cfg/default
```

* copy pxelinux.0 to the same structure (package _syslinux_ needs to be installed)

```bash
cp /usr/share/syslinux/pxelinux.0 /srv/tftpboot/legacy/x86_64/
```

* edit the configuration to get all the correct boot options in place.  Start with editing */srv/tftpboot/legacy/x86_64/pxelinux.cfg/default*:

```yaml
default install

# hard disk
label harddisk
  localboot -2
# install
label install
  kernel linux
  append initrd=initrd showopts install=http://192.168.122.1/provisioning/x86_64/sles12sp3/cd1 textmode=1 autoyast=http://192.168.122.1/autoyast/autoyast_sles12sp3_ses5.xml
  #autoyast=http://192.168.122.1/autoyast/autoyast.xml
  #ssh=1 sshpassword=password console=ttyS0,115200
  #autoyast=http://192.168.122.1/provisioning/autoyast/sles12sp3_ceph.xml

display message
implicit 0
prompt 1
timeout 200
```

* edit */srv/tftpboot/legacy/x86_64//message* to reflect the *default* file:

```bash


       ##################  PXE boot menu  ##################



To start the installation enter 'install' and press <return>.


Available boot options:

  install    - Installation (this is default)
  harddisk   - Boot from Hard Disk
```

* set up x86_64 UEFI boot environment

  * install _shim_ package with all its dependencies

```bash
zypper in -y shim
```

* copy directory with efi modules

```bash
rsync -avP /usr/lib/grub2/x86_64-efi /srv/tftpboot/
```

* copy files required for UEFI booting of a grub2-efi environment

```bash
cd /srv/tftpboot/EFI
cp -a /usr/lib64/efi/shim-sles.efi .
cp -a /usr/lib64/efi/MokManager.efi .
cp -a /usr/lib/grub2/x86_64-efi/grub.efi .
cp -a /srv/install/x86_64/sles12sp3/cd1/EFI/BOOT/bootx64.efi .
cp -a /srv/install/x86_64/sles12sp3/cd1/boot/x86_64/loader/linux .
cp -a /srv/install/x86_64/sles12sp3/cd1/boot/x86_64/loader/initrd .
cp -a /usr/lib/grub2/x86_64-efi/grub.efi ../
```

* create */srv/tftpboot/grub.cfg*

```bash
set timeout=5
menuentry 'Install SLES12 SP3 for x86_64' {
 linuxefi /EFI/linux install=http://192.168.122.1/provisioning/x86_64/sles12sp3/cd1 textmode=1 autoyast=http://192.168.122.1/autoyast/autoyast_uefi.xml
# linuxefi /EFI/linux install=http://192.168.122.1/provisioning/x86_64/sles12sp3/cd1 textmode=1 ssh=1 sshpassword=sles
 initrdefi /EFI/initrd
}
```

### AutoYaST 

* create directory */srv/www/htdocs/autoyast* 

* place there autoyast files:

```bash 
/srv/www/htdocs/autoyast/autoyast_sles12sp3_ses5.xml
/srv/www/htdocs/autoyast/autoyast_uefi.xml
```

* `specific step needs to be done for registering SES5`:

```yaml
  <scripts>
    <init-scripts config:type="list">
      <script>
        <debug config:type="boolean">true</debug>
        <feedback config:type="boolean">false</feedback>
        <feedback_type/>
        <filename>activate_ses.sh</filename>
        <interpreter>shell</interpreter>
        <location><![CDATA[]]></location>
        <network_needed config:type="boolean">false</network_needed>
        <notification/>
        <param-list config:type="list"/>
        <source><![CDATA[#!/bin/bash
until [ ! -s /var/run/zypp.pid ]
do
sleep 10
done
SUSEConnect -p ses/5/x86_64
rpm --import http://admin-server.sestest/autoyast/ses5_content.key
zypper ref
zypper up -ly
zypper in -ly salt-minion
systemctl enable salt-minion
echo "master: admin-server.sestest" > /etc/salt/minion.d/master.conf
systemctl start salt-minion
exit 0]]></source>
      </script>
    </init-scripts>
  </scripts>
```

### APACHE

* set up alias for our install directory in file */etc/apache2/conf.d/inst_server.conf*:

```bash
# httpd configuration for Installation Server included by httpd.conf
<IfDefine inst_server>
    Alias /provisioning/ /srv/install/
    <Directory /srv/install/>

        Options +Indexes +FollowSymLinks
        IndexOptions +NameWidth=*

        Require all granted
    </Directory>
</IfDefine>
```

* */etc/apache2/listen.conf*

```bash
Listen 80


<IfDefine SSL>
    <IfDefine !NOSSL>
        <IfModule mod_ssl.c>

            Listen 443

        </IfModule>
    </IfDefine>
</IfDefine>
```
* enable and start apache2 daemon
