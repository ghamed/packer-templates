#!/bin/sh
echo ""
echo "Remove Linux headers and yum cleanup"
yum -y remove gcc kernel-devel kernel-headers
yum -y clean all

echo "Remove Virtualbox specific files"
rm -rf /usr/src/vboxguest* /usr/src/virtualbox-ose-guest*
rm -rf *.iso *.iso.? /tmp/vbox /home/vagrant/.vbox_version

echo "Cleanup log files"
find /var/log -type f | while read f; do echo -ne '' > $f; done;

echo "Network Stuff"
echo "Ensure we get a DHCP address in 4 interfaces"
echo -e "DEVICE=eth0\nONBOOT=yes\nBOOTPROTO=dhcp" > /etc/sysconfig/network-scripts/ifcfg-eth0
echo -e "DEVICE=eth1\nONBOOT=yes\nBOOTPROTO=dhcp" > /etc/sysconfig/network-scripts/ifcfg-eth1
echo -e "DEVICE=eth2\nONBOOT=yes\nBOOTPROTO=dhcp" > /etc/sysconfig/network-scripts/ifcfg-eth2
echo -e "DEVICE=eth3\nONBOOT=yes\nBOOTPROTO=dhcp" > /etc/sysconfig/network-scripts/ifcfg-eth3

echo "Network Cleanup"
rm -f /etc/udev/rules.d/70-persistent-net.rules
sed -i '/^UUID/d'   /etc/sysconfig/network-scripts/ifcfg-enp0s3
sed -i '/^HWADDR/d' /etc/sysconfig/network-scripts/ifcfg-enp0s3


echo "disable unnecessary services"
chkconfig acpid off 
chkconfig auditd off 
chkconfig blk-availability off 
chkconfig bluetooth off 
chkconfig certmonger off 
chkconfig cpuspeed off 
chkconfig cups off 
chkconfig haldaemon off 
chkconfig ip6tables off 
chkconfig lvm2-monitor off 
chkconfig messagebus off 
chkconfig mdmonitor off 
chkconfig rpcbind off 
chkconfig rpcgssd off 
chkconfig rpcidmapd off 
chkconfig yum-updateonboot off 

echo "Prevent users from pressing CLTR+ALT+DEL in the VNC console that will reboot the host"
rm -rf /etc/init/control-alt-delete.conf
grep -v "^ca::ctrlaltdel:" /etc/inittab > /etc/inittab.new
mv /etc/inittab.new /etc/inittab
rm -rf /etc/init/control-alt-delete.conf

echo '==> Applying slow DNS fix'
if [[ "${PACKER_BUILDER_TYPE}" =~ "virtualbox" ]]; then
  ## https://access.redhat.com/site/solutions/58625 (subscription required)
  # http://www.linuxquestions.org/questions/showthread.php?p=4399340#post4399340
  # add 'single-request-reopen' so it is included when /etc/resolv.conf is generated
  echo 'RES_OPTIONS="single-request-reopen"' >> /etc/sysconfig/network
  service network restart
  echo '==> Slow DNS fix applied (single-request-reopen)'
else
  echo '==> Slow DNS fix not required for this platform, skipping'
fi

echo '==> Configuring sshd_config options'
echo '==> Turning off sshd DNS lookup to prevent timeout delay'
echo "UseDNS no" >> /etc/ssh/sshd_config


echo "==> Cleaning up temporary network addresses"
# Make sure udev doesn't block our network
if grep -q -i "release 6" /etc/redhat-release ; then
  rm -f /etc/udev/rules.d/70-persistent-net.rules
  mkdir /etc/udev/rules.d/70-persistent-net.rules
  rm /lib/udev/rules.d/75-persistent-net-generator.rules
fi
rm -rf /dev/.udev/
if [ -f /etc/sysconfig/network-scripts/ifcfg-eth0 ] ; then
  sed -i "/^HWADDR/d" /etc/sysconfig/network-scripts/ifcfg-eth0
  sed -i "/^UUID/d" /etc/sysconfig/network-scripts/ifcfg-eth0
fi
# new-style network device naming for centos7
if grep -q -i "release 7" /etc/redhat-release ; then
  # radio off & remove all interface configration
  nmcli radio all off
  /bin/systemctl stop NetworkManager.service
  for ifcfg in `ls /etc/sysconfig/network-scripts/ifcfg-* |grep -v ifcfg-lo` ; do
    rm -f $ifcfg
  done
  rm -rf /var/lib/NetworkManager/*
  echo "==> Setup /etc/rc.d/rc.local for CentOS7"
  cat <<_EOF_ | cat >> /etc/rc.d/rc.local
  #BOXCUTTER-BEGIN
  LANG=C
  # delete all connection
  for con in \`nmcli -t -f uuid con\`; do
    if [ "\$con" != "" ]; then
      nmcli con del \$con
    fi
  done
  # add gateway interface connection.
  gwdev=\`nmcli dev | grep ethernet | egrep -v 'unmanaged' | head -n 1 | awk '{print \$1}'\`
  if [ "\$gwdev" != "" ]; then
    nmcli c add type eth ifname \$gwdev con-name \$gwdev
  fi
  sed -i -e "/^#BOXCUTTER-BEGIN/,/^#BOXCUTTER-END/{s/^/# /}" /etc/rc.d/rc.local
  chmod -x /etc/rc.d/rc.local
  #BOXCUTTER-END
  _EOF_
  chmod +x /etc/rc.d/rc.local
fi

