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

## Reference
https://www.cyberciti.biz/faq/how-to-install-kvm-on-ubuntu-20-04-lts-headless-server/  
https://www.answertopia.com/ubuntu/creating-an-ubuntu-kvm-networked-bridge-interface/

