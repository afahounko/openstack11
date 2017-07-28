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

## SSH keys

Create the ssh key for root user:

```
host ~]# ssh-keygen -b 2048 -t rsa -f ~/.ssh/id_rsa -q -N ""
```

The public key will be injected in the cloud image and help to have a access without password.


## DOMAIN NAME

It's recommanded to have a proper DNS server for foward and reverse name resolutions.
We will cover later or in another post how to integrate a DNS service with Identity Manager (IDM) or FreeIPA.
Let's add entries to /etc/hosts to represent it for future ease of access:

```
host ~]# cat << EOF >> /etc/hosts
10.10.0.20   osp-undercloud.local.dc osp-undercloud
EOF
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

- I will use OpenvSwitch for network bridges instead of linux traditional bridges:

```
host ~]# yum install openvswitch 
```

- Enable OpenvSwitch:

```
host ~]# systemctl enable openvswitch 
```

- Start OpenvSwitch:

```
host ~]# systemctl start openvswitch 
```

Let's create two (02) OpenvSwtich (ovs) networks `ovsbr-int` and `ovsbr-ctlplane`.

1. ovsbr-int is the provisioning network `(10.10.0.0/24)`

2. ovsbr-ctlplane is the undercloud pxe network `(192.168.24.0/24)`

### OVS network configuration files:

- ovsbr-int:

```
host ~]# cat << EOF > /etc/sysconfig/network-scripts/ifcfg-ovsbr-int
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

- ovsbr-ctlplane:

```
host ~]# cat << EOF > /etc/sysconfig/network-scripts/ifcfg-ovsbr-ctlplane
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

Restart network configuration:
```
host ~]# systemctl restart network
```

Check ovs bridge status:
```
host ~]# ovs-vsctl show
b51c43b9-7221-404c-b353-d6edc2dc7a41
    Bridge ovsbr-ctlplane
        Port ovsbr-ctlplane
            Interface ovsbr-ctlplane
                type: internal
    Bridge ovsbr-int
        Port ovsbr-int
            Interface ovsbr-int
                type: internal
```

`Note:`
To configure the external ethernet device as ovs bridge (see my previous post)

## Libvirt openvswitch networks

Create Libvirt networks accordingly to the ovs networks created.

Create network definition files:

- ovsbr-int:

```
host ~]# cat << EOF > /tmp/ovsnet-int.xml
<network>
  <name>ovsnet-int</name>
  <forward mode='bridge'/>
  <bridge name='ovsbr-int'/>
  <virtualport type='openvswitch'/>
</network>
EOF
```

- ovsbr-ctlplane:

```
host ~]# cat << EOF > /tmp/ovsnet-ctlplane.xml
<network>
  <name>ovsnet-ctlplane</name>
  <forward mode='bridge'/>
  <bridge name='ovsbr-ctlplane'/>
  <virtualport type='openvswitch'/>
</network>
EOF
```

Register ovs networks:
```
host ~]# virsh net-define /tmp/ovsnet-int.xml
host ~]# virsh net-define /tmp/ovsnet-ctlplane.xml
```

Activate ovs networks:
```
host ~]# virsh net-start ovsnet-int
host ~]# virsh net-start ovsnet-ctlplane
```

Autostart ovs networks:
```
host ~]# virsh net-autostart ovsnet-int
host ~]# virsh net-autostart ovsnet-ctlplane
```

Check ovs network:
```
host ~]# virsh net-list
```

## Firewalld

Enable firewall rule to permit the ovs bridges inter routing and NAT to the external world via `eth0`.

We assume the external network device on the hypervisor is `eth0`:

```
host ~]# firewall-cmd --permanent --direct --add-rule ipv4 nat POSTROUTING 0 -o eth0 -j MASQUERADE

host ~]# firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

host ~]# firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 -i ovsbr-int -o eth0 -m conntrack --ctstate NEW -j ACCEPT

host ~]# firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 -i ovsbr-int -o ovsbr-ctlplane -m conntrack --ctstate NEW -j ACCEPT

host ~]# firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 0 -i ovsbr-ctlplane -o ovsbr-int -m conntrack --ctstate NEW -j ACCEPT
```
Reload firewall rules:
```
host ~]# firewall-cmd --reload
```

Check firewalld rules:
```
host ~]# firewall-cmd --direct --get-all-rules
```

## ISC DHCP (Dynamic Host Configuration Protocol)

We will use the ISC DHCP (Dynamic Host Configuration Protocol) server to provide necessary IP leases for ovs ovsbr-int network.

`Note`: ovsbr-ctlplane network will be manage by the undercloud PXE   

- Install ISC DHCP:

```
host ~]# yum install -y dhcp
```

- Configure /etc/dhcp/dhcpd.conf:

```
host ~]# cat << EOF > /etc/dhcp/dhcpd.conf
# Default domain
option domain-name "local.dc";
# name servers
option domain-name-servers 8.8.8.8;

default-lease-time 600;
max-lease-time 7200;

#ddns-update-style none;
#authoritative;
log-facility local7;

# -- default-vibr0
subnet 192.168.122.0 netmask 255.255.255.0 {
}

# -- ovsbr-ctlplane
subnet 192.168.24.0 netmask 255.255.255.0 {
}

# -- ovsbr-int
subnet 10.10.0.0 netmask 255.255.255.0 {
  option routers 10.10.0.1;
  range 10.10.0.50 10.10.0.100;
}

# -- fixed ip for osp-undercloud
host osp-undercloud {
  hardware ethernet 52:54:00:5c:23:43;
  fixed-address 10.10.0.20;
}

EOF
```

- Enable dhcp service:

```
host ~]# systemctl enable dhcpd.service
```

- Start dhcp service:

```
host ~]# systemctl start dhcpd.service
```

## IPMI access

Create access on the Hypervisor so undercloud (ironic) can control virtual machines deployed on the KVM.

- Create user account stack:
```
host ~]# useradd stack
host ~]# echo "RedHatOSP11" | passwd stack --stdin
```

- Grant manage privileges to user stack:

```
host ~]# cat << EOF > /etc/polkit-1/localauthority/50-local.d/50-libvirt-user-stack.pkla
[libvirt Management Access]
Identity=unix-user:stack
Action=org.libvirt.unix.manage
ResultAny=yes
ResultInactive=yes
ResultActive=yes
EOF
```


## RedHat KVM guest image

### Download the image

Download Red Hat guest image `rhel-guest-image-7.3-36.x86_64.qcow2`  from Red Hat download page and save it in `/var/lib/libvirt/images/`.

```
host ~]# qemu-img info /var/lib/libvirt/images/rhel-guest-image-7.3-36.x86_64.qcow2
image: /var/lib/libvirt/images/rhel-guest-image-7.3-36.x86_64.qcow2
file format: qcow2
virtual size: 10G (10737418240 bytes)
disk size: 547M
cluster_size: 65536
Format specific information:
    compat: 0.10
```

### Resize the cloud image

You need to install a set of tools to interact with the cloud image:

```
host ~]# yum install -y libguestfs-tools libguestfs-xfs qemu-img
```

Check Red Hat kvm cloud image actual size:
```
host ~]# virt-filesystems --long -h --all -a /var/lib/libvirt/images/rhel-guest-image-7.3-36.x86_64.qcow2

Name       Type        VFS  Label  MBR  Size  Parent
/dev/sda1  filesystem  xfs  -      -    7.8G  -
/dev/sda1  partition   -    -      83   7.8G  /dev/sda
/dev/sda   device      -    -      -    10G   -
[root@sd-73122 images]#
```

Resize the kvm cloud image to `60GB`:
```
host ~]# qemu-img resize /var/lib/libvirt/images/rhel-guest-image-7.3-36.x86_64.qcow2 60G
```

Confirm Red Hat kvm cloud image size after resizing:
```
host ~]# virt-filesystems --long -h --all -a /var/lib/libvirt/images/rhel-guest-image-7.3-36.x86_64.qcow2

Name       Type        VFS  Label  MBR  Size  Parent
/dev/sda1  filesystem  xfs  -      -    60G   -
/dev/sda1  partition   -    -      83   60G   /dev/sda
/dev/sda   device      -    -      -    60G   -
```


## Undercloud image deployment


### Create undercloud image

- Create Undercloud image:

```
host ~]# cd /var/lib/libvirt/images/

host ~]# qemu-img create -f qcow2 -b rhel-guest-image-7.3-36.x86_64.qcow2 osp-undercloud.qcow2
```

### Customize undercloud image

- By default the guest image comes with a random root password. Let's set the root password to something we know **RedHat4ever**: 

```
host ~]# virt-customize -a osp-undercloud.qcow2 --root-password password:RedHat4ever
```

- Inject root user public key (id_rsa.pub): 

```
host ~]# virt-customize -a osp-undercloud.qcow2 --ssh-inject root
```

- Define the hostname: 

```
host ~]# virt-customize -a osp-undercloud.qcow2 --hostname osp-undercloud.local.dc
```

- Remove cloud-init to avoid unnecessary lookup during booting process:

```
host ~]# virt-customize -a osp-undercloud.qcow2 --uninstall cloud-init
```

- Update the cloud image packages:

```
host ~]# virt-customize -a osp-undercloud.qcow2 --update
```

- Create network configuration files (eth0 & eth1) for `ovsbr-ctlplane` and `ovsbr-int`:

```
host ~]# virt-customize -a osp-undercloud.qcow2 --run-command 'cp /etc/sysconfig/network-scripts/ifcfg-eth{0,1} && sed -i s/DEVICE=.*/DEVICE=eth1/g /etc/sysconfig/network-scripts/ifcfg-eth1'
```

- Disable the automatic boot on eth0 (ovsbr-ctlplane):

```
host ~]# virt-customize -a osp-undercloud.qcow2 --run-command 'sed -i s/ONBOOT=.*/ONBOOT=no/g /etc/sysconfig/network-scripts/ifcfg-eth0'
```

- Enable the automatic on eth1 (ovsbr-int):

```
host ~]# virt-customize -a osp-undercloud.qcow2 --run-command 'sed -i s/ONBOOT=.*/ONBOOT=yes/g /etc/sysconfig/network-scripts/ifcfg-eth1'
```

### Install undercloud image

- Create virtual machine for the undercloud image with our two networks (ovsbr-int and ovsbr-ctlplane): 

```
host ~]# virt-install --import --name osp-undercloud --ram 16384 --vcpus 4 --disk /var/lib/libvirt/images/osp-undercloud.qcow2,format=qcow2,bus=virtio  --network bridge=ovsbr-ctlplane,model=virtio,virtualport_type=openvswitch  --network bridge=ovsbr-int,model=virtio,virtualport_type=openvswitch,mac=52:54:00:5c:23:43  --os-type=linux --os-variant=rhel7.3 --graphics none --autostart --noautoconsole


Starting install...
Creating domain... 
```

This step will create the `osp-undercloud` domain with 16GB of memory, 4 cpu with two nic connected respectively to ovsbr-ctlplane and ovsbr-int.

`Note`:
Adding the mac address (52:54:00:5c:23:43) in the setup command will create the nic device with the specified hardware address. The connected nic interface will grab the ip address filled in the dhcp config file (10.10.0.20).

Verify connectivity to this machine, with it not requiring a password:

```
host ~]#  ssh root@osp-undercloud
osp-undercloud]# exit
host ~]#
```

`Note:`
If for a reason you did not inject the root public key during the undercloud cloud image customization, you can still copy the root key with ssh-copy-id:

```
host ~]#  ssh-copy-id -i ~/.ssh/id_rsa.pub root@osp-undercloud
```

### Snapshot the Undercloud VM

Lets create a snapshot of our undercloud virtual machine in the case we run into any problems; this will allow us to revert back to a working state without having to build up our environment from scratch:

```
host ~]#  virsh snapshot-create-as osp-undercloud osp-undercloud-snap1
Domain snapshot osp-undercloud-snap1 created

host ~]# virsh snapshot-list osp-undercloud
 Name                 Creation Time             State
------------------------------------------------------------
 osp-undercloud-snap1 2017-07-27 10:25:24 +0200 running
```

If you ever need to restore the undercloud to the current state, you can execute the following command:

```
host ~]# virsh snapshot-revert --domain osp-undercloud <snapshot-name>
```

## Create nodes for the overcloud
We will provision five (05) nodes for the overcloud setup: 
- 3 x controller nodes (for HA)
- 2 x compute nodes

- Create 60GB thin-provisioned disks for the five guests:

```
host ~]# cd /var/lib/libvirt/images/
host ~]# for i in {1..5}; do qemu-img create -f qcow2 \
    -o preallocation=metadata osp-overcloud-node$i.qcow2 60G; done
```

Let's now create libvirt definitions for our overcloud virtual machines, ensuring the following characteristics:

- 4 vCPU's
- 8GB memory
- 60GB disk (we just formatted)
- 1 x nic ovsbr-ctlplane
- 1 x nic ovsbr-int

```
host ~]# for i in {1..5}; do \
    virt-install --ram 8192 --vcpus 4 --os-variant rhel7 \
    --disk path=/var/lib/libvirt/images/osp-overcloud-node$i.qcow2,device=disk,bus=virtio,format=qcow2 \
    --noautoconsole --vnc --network bridge=ovsbr-ctlplane,model=virtio,virtualport_type=openvswitch \
    --network bridge=ovsbr-int,model=virtio,virtualport_type=openvswitch --name osp-overcloud-node$i \
    --cpu SandyBridge,+vmx \
    --dry-run --print-xml > /tmp/osp-overcloud-node$i.xml; \
    virsh define --file /tmp/osp-overcloud-node$i.xml; done
```

`NOTE`: We add in the flag "--cpu SandyBridge,+vmx" here to make sure that our overcloud nodes enable nested virtualisation.

We should now have a number of additional virtual machines configured, but not started:

```
host ~]# virsh list --all
 Id    Name                           State
----------------------------------------------------
 1     osp-undercloud                     running
 -     osp-overcloud-node1                shut off
 -     osp-overcloud-node2                shut off
 -     osp-overcloud-node3                shut off
 -     osp-overcloud-node4                shut off
 -     osp-overcloud-node5                shut off
```

`NOTE:` Do NOT start these machines; we'll have OSP director manage the power status of these machines going forward.

## Undercloud installation

- Connect to the `osp-undercloud` virtual machine:

```
host ~]#  ssh root@osp-undercloud
```

- Set hostname:

```
osp-undercloud ~]# hostnamectl set-hostname osp-undercloud.local.dc
osp-undercloud ~]# hostnamectl set-hostname --transient osp-undercloud.local.dc
```

- Set /etc/hosts file:

```
osp-undercloud ~]# cat << EOF > /etc/hosts
# The following lines are desirable for IPv4 capable hosts
127.0.0.1 $(hostname -f) $(hostname -s)
127.0.0.1 localhost.localdomain localhost
127.0.0.1 localhost4.localdomain4 localhost4

# The following lines are desirable for IPv6 capable hosts
::1 $(hostname -f) $(hostname -s)
::1 localhost.localdomain localhost
::1 localhost6.localdomain6 localhost6

EOF
```

- Create user stack:

```
osp-undercloud ~]# useradd stack
osp-undercloud ~]# echo "RedHatOSP11" | passwd stack --stdin
```

- Allow stack to sudo without password:

```
osp-undercloud ~]# echo "stack ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/stack
osp-undercloud ~]# chmod 0440 /etc/sudoers.d/stack
osp-undercloud ~]# su - stack
osp-undercloud ~]$ whoami
stack
```

- Create images and template folders:

```
osp-undercloud ~]$ mkdir ~/images
osp-undercloud ~]$ mkdir ~/templates
```

- Register the system: 

```
osp-undercloud ~]$ sudo subscription-manager register
```

- Find the entitlement pool ID for Red Hat OpenStack Platform director

```
osp-undercloud ~]$ sudo subscription-manager list --available --all --matches="*OpenStack*"
Subscription Name:   Name of SKU
Provides:            Red Hat Single Sign-On
                     Red Hat Enterprise Linux Workstation
                     Red Hat CloudForms
                     Red Hat OpenStack
                     Red Hat Software Collections (for RHEL Workstation)
                     Red Hat Virtualization
SKU:                 SKU-Number
Contract:            Contract-Number
Pool ID:             Valid-Pool-Number-123456
Provides Management: Yes
Available:           1
Suggested:           1
Service Level:       Support-level
Service Type:        Service-Type
Subscription Type:   Sub-type
Ends:                End-date
System Type:         Physical
```

- Attach Red Hat OpenStack Platform director pool ID:

```
osp-undercloud ~]$ sudo subscription-manager attach --pool=Valid-Pool-Number-123456
```

- Disable all default repositories, and then enable the required Red Hat Enterprise Linux repositories:

```
osp-undercloud ~]$ sudo subscription-manager repos --disable=*
osp-undercloud ~]$ sudo subscription-manager repos --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms --enable=rhel-ha-for-rhel-7-server-rpms --enable=rhel-7-server-openstack-11-rpms
```

- Perform an update on your system to make sure you have the latest base system packages:

```
osp-undercloud ~]$ sudo yum update -y
osp-undercloud ~]$ sudo reboot
```

### Install Director packages

```
osp-undercloud ~]# yum install -y python-tripleoclient
```

### Director configuration

Director configuration will be performed as stack user:

```
osp-undercloud ~]# su - stack
```

- undercloud config file:

```
osp-undercloud ~]$ cat << EOF > ~/undercloud.conf
[DEFAULT]
undercloud_hostname = $(hostname -f)
local_ip = 192.168.24.1/24
network_gateway = 192.168.24.1
undercloud_public_vip = 192.168.24.2
undercloud_admin_vip = 192.168.24.3
local_interface = eth0
network_cidr = 192.168.24.0/24
masquerade_network = 192.168.24.0/24
dhcp_start = 192.168.24.5
dhcp_end = 192.168.24.24
discovery_interface = br-ctlplane
discovery_iprange = 192.168.24.100,192.168.24.120
enabled_drivers = pxe_ipmitool,pxe_drac,pxe_ilo,pxe_ssh
[auth]
EOF
```

`Note`: The complete template of undercloud configuration file is located here `/usr/share/instack-undercloud/undercloud.conf.sample`

- Run undercloud setup:

```
osp-undercloud ~]$ openstack undercloud install
```

The configuration script generates two files when complete:

1. `undercloud-passwords.conf` - A list of all passwords for the director’s services.
2. `stackrc` - A set of initialization variables to help you access the director’s command line tools.


The configuration also starts all OpenStack Platform services automatically. 

- Check the enabled services using the following command:

```
osp-undercloud ~]$ sudo systemctl list-units openstack-*
```

- To initialize the stack user to use the command line tools, run the following command:

```
osp-undercloud ~]$ source ~/stackrc
```

- You can now use the director’s command line tools.

```
osp-undercloud ~]$ source ~/stackrc
```


### Images for overcloud nodes

The director requires several disk images for provisioning overcloud nodes. This includes:

1. An introspection kernel and ramdisk - Used for bare metal system introspection over PXE boot.
2. A deployment kernel and ramdisk - Used for system provisioning and deployment.
3. An overcloud kernel, ramdisk, and full image - A base overcloud system that is written to the node’s hard disk.

- Obtain these images from the rhosp-director-images and rhosp-director-images-ipa packages:

```
osp-undercloud ~]$ sudo yum install rhosp-director-images rhosp-director-images-ipa
```

- Extract the archives to the images directory on the stack user’s home (/home/stack/images):

```
osp-undercloud ~]$ cd ~/images
osp-undercloud ~]$ for i in /usr/share/rhosp-director-images/overcloud-full-latest-11.0.tar /usr/share/rhosp-director-images/ironic-python-agent-latest-11.0.tar; do tar -xvf $i; done
```

- Import these images into the director:

```
osp-undercloud ~]$ openstack overcloud image upload --image-path /home/stack/images/
```

This uploads the following images into the director: bm-deploy-kernel, bm-deploy-ramdisk, overcloud-full, overcloud-full-initrd, overcloud-full-vmlinuz. These are the images for deployment and the overcloud. The script also installs the introspection images on the director’s PXE server.

- Check uploaded images:

```
osp-undercloud ~]$ openstack image list
```

- Check subnet:

```
osp-undercloud ~]$ openstack subnet list
```

- Define the name server for the subnet:

```
osp-undercloud ~]$ openstack subnet set --dns-nameserver 8.8.8.8 --dns-nameserver 8.8.4.4 $(openstack subnet list | awk '$4 == "ctlplane-subnet" {print $2};')
```

- Check subnet after update:

```
osp-undercloud ~]$ openstack subnet show $(openstack subnet list | awk '$4 == "ctlplane-subnet" {print $2};')
```


## Registering nodes for the overcloud















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
