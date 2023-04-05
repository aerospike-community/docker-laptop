## Getting ready
* the file `settings.sh` should have `SERIAL=0` set
* the file `settings.sh` should have `CDISO` set to a valid installation media
* run `qcow-data-reset.sh`
* run `qcow-root-delete.sh 8G`
* run `cp images/root/amd64/ro.qcow2 images/root/amd64/rw.qcow2`
* run `start.sh`

## Installation

### AMD64

Install the minimal base debian system with no added utilities. Disk `sda` should have 1 partition occupying whole disk for the root filesystem. Disk `sdb` should not be partitioned for now. Set all timezones to UTC, and all username/password to `docker`.

## ARM64

Download a prebuilt nocloud qcow2 image from https://cloud.debian.org/images/cloud/bullseye/latest/ and put it in root/rw.qcow2

When running, after logging in with root with no password, remember to set a password using `passwd`

## Setup basics - amd64 only
```
apt-get purge linux-image-5.10.0-20-amd64
apt-get -y install cloud-guest-utils openssh-server sudo
usermod -a -G sudo docker
```

SSH in from Mac to `docker@192.168.105.2`, and run `sudo -i`

Configure networking:

```
cat <<'EOF' > /etc/systemd/network/enp.network
[Match]
Name=enp*

[Network]
DHCP=yes
EOF
cat <<'EOF' > /etc/systemd/network/eth.network
[Match]
Name=eth*

[Network]
DHCP=yes
EOF
cat <<'EOF' > /etc/network/interfaces
source /etc/network/interfaces.d/*
auto lo
iface lo inet loopback
EOF
systemctl enable systemd-networkd
```

## Configure grub:

```
echo "GRUB_TERMINAL=serial" >> /etc/default/grub

#amd64
echo "GRUB_CMDLINE_LINUX=console=ttyS0" >> /etc/default/grub
#arm64
echo "GRUB_CMDLINE_LINUX=console=ttyAMA0" >> /etc/default/grub

echo "GRUB_TIMEOUT=1" >> /etc/default/grub
update-grub2
poweroff
```

Switch `settings.sh` to use `SERIAL=1` now. Run `start.sh` and on the `Waiting for docker to come up` prompt, press `CTRL+c`.

## Configuring the image

### Docker

```
ssh docker@192.168.105.2
sudo -i
apt-get update
apt-get -y install ca-certificates curl gnupg lsb-release
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
apt-get -y install docker-ce docker-ce-cli containerd.io docker-compose-plugin ntpdate ntp
mkdir -p /usr/lib/systemd/system/docker.service.d
cat <<'EOF' > /usr/lib/systemd/system/docker.service.d/override.conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H fd:// --containerd=/run/containerd/containerd.sock
EOF
systemctl daemon-reload
```

### HomePath Sharing

```
cat <<'EOF' > /usr/local/bin/dockerlaptop-shares.sh
[ "${1}" != "start" ] && exit 0
modprobe virtio_input || exit 1
modprobe virtio_pci || exit 1
modprobe 9pnet || exit 1
modprobe 9pnet_virtio || exit 1
modprobe 9p || exit 1
MHOME=$(cat /sys/firmware/qemu_fw_cfg/by_name/opt/local.dockerlaptop.homepath/raw)
[ "${MHOME}" = "" ] && echo "no home passed in firmware" && exit 1
mkdir -p ${MHOME}
mount -t 9p -o trans=virtio,msize=1048576,version=9p2000.u HostHome ${MHOME}
dhclient
EOF
chmod 755 /usr/local/bin/dockerlaptop-shares.sh
cat <<'EOF' > /etc/systemd/system/dockerlaptop-shares.service
[Unit]
Description=DockerLaptop Shares
Before=docker.service
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/bash /usr/local/bin/dockerlaptop-shares.sh start
RemainAfterExit=true
ExecStop=/bin/bash /usr/local/bin/dockerlaptop-shares.sh stop
StandardOutput=journal

[Install]
WantedBy=multi-user.target
EOF
systemctl enable dockerlaptop-shares
```

### Docker volume from sdb

NOTE ON ARM: for arm is should be `vdb` nor `sdb`

```
cat <<'EOF' > /usr/local/bin/dockerlaptop-dockervol.sh
[ "${1}" != "start" ] && exit 0
function makedev() {
	ls /dev/sdb || return 1
	blkid /dev/sdb && return 0
	mkfs.ext4 -L "dockervol" /dev/sdb || return 1
	mount /dev/sdb /mnt || return 1
	mv /var/lib/docker/* /mnt/
	umount /mnt
}
function fixfstab() {
	grep dockervol /etc/fstab && return 0
	echo "/dev/disk/by-label/dockervol /var/lib/docker ext4 auto,nofail 0 2" >> /etc/fstab
	mount -a || return 1
}
echo "entering makedev"
makedev || exit 1
echo "resizing"
e2fsck -n -f /dev/sdb
resize2fs /dev/sdb
echo "entering fixfstab"
fixfstab || exit 1
EOF
chmod 755 /usr/local/bin/dockerlaptop-dockervol.sh
cat <<'EOF' > /etc/systemd/system/dockerlaptop-dockervol.service
[Unit]
Description=DockerLaptop DockerVol
Before=docker.service
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/bash /usr/local/bin/dockerlaptop-dockervol.sh start
RemainAfterExit=true
ExecStop=/bin/bash /usr/local/bin/dockerlaptop-dockervol.sh stop
StandardOutput=journal

[Install]
WantedBy=multi-user.target
EOF
systemctl enable dockerlaptop-dockervol
```

### Wait for docker

NOTE ON ARM: ttyAMA0 instead of ttyS0

```
cat <<'EOF' > /usr/local/bin/dockerlaptop-dockerwait.sh
[ "${1}" != "start" ] && exit 0
UUID=$(cat /sys/firmware/qemu_fw_cfg/by_name/opt/local.dockerlaptop.uid/raw)
MYIP=$(hostname -I |sed 's/ /\n/g' |egrep -v '^172|^127|:' |egrep '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+')
echo "${UUID} IP=${MYIP}" > /dev/ttyS0
while true
do
ip addr sh |grep docker >/dev/null 2>&1
if [ $? -eq 0 ]
then
  sleep 1
  iptables -I FORWARD -j ACCEPT
  iptables -P FORWARD ACCEPT
  iptables -I DOCKER-USER -j ACCEPT
  echo "${UUID} DOCKERLAPTOP-RUNNING" > /dev/ttyS0
  exit 0
fi
sleep 1
done
EOF
chmod 755 /usr/local/bin/dockerlaptop-dockerwait.sh
cat <<'EOF' > /etc/systemd/system/dockerlaptop-dockerwait.service
[Unit]
Description=DockerLaptop Docker Wait
After=docker.service

[Service]
Type=oneshot
ExecStart=/bin/bash /usr/local/bin/dockerlaptop-dockerwait.sh start
RemainAfterExit=true
ExecStop=/bin/bash /usr/local/bin/dockerlaptop-dockerwait.sh stop
StandardOutput=journal

[Install]
WantedBy=multi-user.target
EOF
systemctl enable dockerlaptop-dockerwait
```

### Configure autologin - amd64

```
mkdir -p /etc/systemd/system/serial-getty@ttyS0.service.d
cat <<'EOF' > /etc/systemd/system/serial-getty@ttyS0.service.d/override.conf
[Service]
ExecStart=
ExecStart=-/sbin/agetty -o '-p -- \\u' --keep-baud 115200,38400,9600 --noclear --autologin root ttyS0 $TERM
EOF
systemctl daemon-reload

echo "auth sufficient pam_listfile.so item=tty sense=allow file=/etc/securetty onerr=fail apply=root" > /etc/pam.d/login1; cat /etc/pam.d/login >> /etc/pam.d/login1; mv /etc/pam.d/login1 /etc/pam.d/login

echo ttyS0 > /etc/securetty
```

### Configure autologin - arm64

```
mkdir -p /etc/systemd/system/serial-getty@ttyAMA0.service.d
cat <<'EOF' > /etc/systemd/system/serial-getty@ttyAMA0.service.d/override.conf
[Service]
ExecStart=
ExecStart=-/sbin/agetty -o '-p -- \\u' --keep-baud 115200,38400,9600 --noclear --autologin root ttyAMA0 $TERM
EOF
systemctl daemon-reload

echo "auth sufficient pam_listfile.so item=tty sense=allow file=/etc/securetty onerr=fail apply=root" > /etc/pam.d/login1; cat /etc/pam.d/login >> /etc/pam.d/login1; mv /etc/pam.d/login1 /etc/pam.d/login

echo ttyAMA0 > /etc/securetty
```

### Upgrader

```
cat <<'EOF' > /usr/local/bin/upgrade.sh
apt-get update
apt-get -y dist-upgrade
echo -e '\n\nUPGRADE COMPLETED, PRESS CTRL+C TO RETURN\nONCE IN THE MENU, WE SUGGEST COMMITING THE NEW ROOT DISK IMAGE'
EOF
chmod 755 /usr/local/bin/upgrade.sh
```

### MTU and NTP workarounds

```
cat <<'EOF' >> /etc/crontab
@reboot         root    /usr/local/bin/mtu.sh
* *     * * *   root    ntpdate -s ntp.ubuntu.com
EOF
cat <<'EOF' > /usr/local/bin/mtu.sh
IFACE=""
while [ "${IFACE}" = "" ]
do
  sleep 1
  IFACE=$(ip route ls |egrep '^default' |awk '{print $NF}')
done
ip link set dev ${IFACE} mtu 1492
EOF
```

## Shrink image

```
service docker stop
cd /var/cache/apt/archives
rm -rf 
cd /var/log
rm -rf *
cd /var/lib/apt/lists
rm -rf *
cd /var/cache/apt
rm -rf *bin
fstrim -av
poweroff
```

On Mac:
```
cd images/root/amd64
qemu-img convert -f qcow2 -O qcow2 rw.qcow2 rw-new.qcow2
mv rw-new.qcow2 rw.qcow2
```

## Packaging

cd docker-laptop

### Cleanup

```
rm -rf ../docker-laptop-pkg
mkdir -p ../docker-laptop-pkg/v1
rm -f images/data/*/*.qcow2
rm -f images/root/*/rw.qcow2
sudo rm -f log/*
```

### Images

```
mkdir -p ../docker-laptop-pkg/v1/images/root/amd64
mkdir -p ../docker-laptop-pkg/v1/images/root/arm64

cd images/root/amd64
gzip ro.qcow2
cd ../arm64
gzip ro.qcow2
cd ../../..
mv images/root/amd64/ro.qcow2.gz ../docker-laptop-pkg/v1/images/root/amd64/
mv images/root/arm64/ro.qcow2.gz ../docker-laptop-pkg/v1/images/root/arm64/
```

### Main

```
cd ..
cp -a docker-laptop .docker-laptop
tar -zcvf docker-laptop-pkg/docker-laptop.tgz .docker-laptop
rm -rf .docker-laptop
cd docker-laptop
```

### Scripts

```
tar -zcvf ../docker-laptop-pkg/v1/scripts.tgz bin etc/version.sh
```
