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

## Libvirt (KVM) Installation

Installation of libvirt KVM is quite straigth forward action:
```
host ~]# yum install -y libvirt ...  
```

Start and enable libvirt
```
host ~]# systemctl enable libvirtd  
```


## Enable nested virtualization



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
