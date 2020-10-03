## KVM

## Install packages
```
$ sudo apt install qemu-kvm libvirt-daemon-system \
    libvirt-clients virtinst cpu-checker libguestfs-tools libosinfo-bin
$ ip a s virbr0
```

## Create bridge network
```
$ sudo vim /etc/netplan/00-installer-config.yaml
```

* /etc/netplan/00-installer-config.yaml
```
network:
  ethernets:
    eno1:
      dhcp4: no
      dhcp6: no
  bridges:
    br0:
      interfaces: [eno1]
      dhcp4: no
      addresses: [192.168.1.XXX/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [192.168.1.1, 8.8.8.8, 8.8.4.4]
```

```
$ netplan apply -a
or
$ sudo shutdown -r now
```

## Modify kernel parameter

```
$ sudo vim /etc/sysctl.d/90-bridge-network.conf
```

Add kernel parameters like below.

```
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0
```

* /etc/udev/rules.d/90-bridge-network.rules
```
ACTION=="add", SUBSYSTEM=="module", KERNEL=="br_netfilter", \
    RUN+="/lib/systemd/systemd-sysctl --prefix=/net/bridge"
```

Run systemd-sysctl to load new kernel parameters.

```
/lib/systemd/systemd-sysctl --prefix=/net/bridge
```

## Install guest os

Download ubuntu server iso and prepare to launch kvm.

```
$ wget https://releases.ubuntu.com/20.04/ubuntu-20.04.1-live-server-amd64.iso

$ sudo mkdir -p /var/kvm/iso
$ sudo mv ~/ubuntu-20.04.1-live-server-amd64.iso /var/kvm/iso
```

Create the script.

```
$ sudo vim /var/kvm/iso/install_ubuntu20.04.sh
```

* /var/kvm/iso/install_ubuntu20.04.sh
```
#!/usr/bin/env bash

DOMAIN_NAME="ubuntu_$(uname -n)"
mkdir -p /var/kvm/distros/${DOMAIN_NAME}

virt-install \
    --name ${DOMAIN_NAME} \
    --connect=qemu:///system \
    --vcpus=2 \
    --memory 8192 \
    --disk path=/var/kvm/distros/${DOMAIN_NAME}/disk.img,size=40,format=qcow2 \
    --os-type linux \
    --os-variant ubuntu20.04 \
    --arch x86_64 \
    --network bridge:br0 \
    --cdrom /var/kvm/iso/ubuntu-20.04.1-live-server-amd64.iso \
    --graphics vnc,port=15901,listen=127.0.0.1,keymap=us,password=changeme
```

```
$ sudo chmod u+x /var/kvm/iso/install_ubuntu20.04.sh
```

Run install script.

```
$ sudo /var/kvm/iso/install_ubuntu20.04.sh
```


## Reference
https://www.cyberciti.biz/faq/how-to-install-kvm-on-ubuntu-20-04-lts-headless-server/  
https://www.answertopia.com/ubuntu/creating-an-ubuntu-kvm-networked-bridge-interface/

