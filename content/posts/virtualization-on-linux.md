---
title: "virtualization on linux"

date: "2023-10-13"
tags:
- linux
- virtualization
---

Beginner's guide to setup VMs on Linux, without the pain of VirtualBox

<!--more-->

# Why?

- Virtualbox's installation is more tiresome
- Virutalbox's CLI looks more complex. Check it
  [here](https://www.oracle.com/technical-resources/articles/it-infrastructure/admin-manage-vbox-cli.html)

# Resort: libvirt + virt-install

- They are open-source and free
- There is also a GUI based on it, called **virt-manager**
- Unlike virtualbox, you can find installation guides easily from major distro's
  official site:
  - [Ubuntu](https://manpages.ubuntu.com/manpages/trusty/man1/virt-install.1.html#examples)
  - [Fedora](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-guest_virtual_machine_installation_overview-creating_guests_with_virt_install)
  - [Arch](https://wiki.archlinux.org/title/libvirt)

# How do I create a new VM in terminal?

- install an OS iso, and in command line

```bash
# here we use ubuntu 22.04.3 as an example
virt-install \
  --connect qemu:///system \
  --name $YOUR_VM_NAME \
  --os-variant ubuntu22.04 \
  --vcpus 2 \
  --ram 2048 \
  --disk size=20 \
  --location /tmp/ubuntu-22.04.3-live-server-amd64.iso,kernel=casper/vmlinuz,initrd=casper/initrd \
  --network network:default \
  --nographics \
  --extra-args "console=ttyS0"
```

- you can check the manual of
  [virt-install](https://linux.die.net/man/1/virt-install) for the meaning of
  each flag

# How do I create a new VM with GUI?

- virt-manager is straightforward. If you have previous experience with VMWare
  Fusion, Parallel Desktop, VirtualBOx, it's easy to start with.
- Here is a redhat blog for how to use it:
  [the blog](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-creating_guests_with_virt_manager)

# VM management

```bash
# list all the VMs (their names) created by virt-install
virsh list --all # this will only show the VMs created by the current user

# turn on a VM which in `shut off` state
virsh start <VM_name>

# gracefully shutdown a VM
virsh shutdown <VM_name>

# forcefully shutdown a VM
virsh destroy <VM_name>

# remove a vm when it's off
virsh undefine <name>
```

- you can find more
  [here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/chap-managing_guest_virtual_machines_with_virsh)
