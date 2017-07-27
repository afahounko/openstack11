## Red Hat OpenStack Platform 11

This document describes how to install Red Hat OpenStack Platform 11 on a single physical host with KVM.

The configuration of the physical host (Hypervisor):
```
Model: DELL® PowerEdge R720
Processor: 2x Intel® Xeon® E5 2620 v2
Architecture: 12 cores 24 threads 2x @2,1 Ghz cache L3 15MB, x64, VT
RAM: 128 Go DDR3 ECC
Hard Drive: 2 x 500Go SSD
```

## Libvirt (KVM) installation

Installation of libvirt KVM is quite straigth forward action:
```
host ~]# yum install -y yum install libvirt qemu-kvm virt-manager virt-install libguestfs-tools  
```

Start and enable libvirt
```
host ~]# systemctl enable libvirtd
host ~]# systemctl start libvirtd
```


## Nested virtualization

Enabling nested KVM will help to have accelerated nested virtualisation:
```
host ~]# cat << EOF > /etc/modprobe.d/kvm_intel.conf
options kvm-intel nested=1
options kvm-intel enable_shadow_vmcs=1
options kvm-intel enable_apicv=1
options kvm-intel ept=1
EOF
```

Disable the rp_filter to allow our virtual machines to communicate:
```
host ~]# cat << EOF > /etc/sysctl.d/98-rp-filter.conf
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.rp_filter = 0
EOF
```

Enable ip_forwarding:
```
host ~]# cat << EOF > /etc/sysctl.d/90-ip_forward-filter.conf
net.ipv4.ip_forward=1
net.ipv6.conf.default.forwarding=1
net.ipv6.conf.all.forwarding=1
EOF
```

Reboot the system to activate all settings:
```
host ~]# reboot
```

## OpenvSwitch for bridges

I will use OpenvSwitch for network bridges instead of linux traditional bridges:
```
host ~]# yum install openvswitch 
```

Enable OpenvSwitch:
```
host ~]# systemctl enable openvswitch 
```

Start OpenvSwitch:
```
host ~]# systemctl start openvswitch 
```

Let's create two (02) OpenvSwtich (ovs) networks `ovsbr-int` and `ovsbr-ctlplane`.

ovsbr-int is the provisioning network `(10.10.0.0/24)`
ovsbr-ctlplane is the undercloud pxe network `(192.168.24.0/24)`

OVS Network configuration files:
ovsbr-int
```
cat << EOF > /etc/sysconfig/network-scripts/ifcfg-ovsbr-int
# -- Interface ovs bridge ovsbr-int
DEVICE=ovsbr-int
ONBOOT=yes
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=static
IPADDR=10.10.0.1
NETMASK=255.255.255.0
HOTPLUG=no
NM_CONTROLLED=no
ZONE=public 
# IPv6
EOF
```

ovsbr-ctlplane
```
cat << EOF > /etc/sysconfig/network-scripts/ifcfg-ovsbr-ctlplane
# -- Interface ovs bridge ovsbr-ctlplane
DEVICE=ovsbr-ctlplane
ONBOOT=yes
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=static
IPADDR=192.168.24.254
NETMASK=255.255.255.0
HOTPLUG=no
NM_CONTROLLED=no
ZONE=public 
# IPv6
EOF
```


### Markdown

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/afahounko/openstack11/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
