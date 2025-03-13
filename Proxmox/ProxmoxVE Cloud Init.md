---
tags:
  - "#proxmox"
  - "#virtualization"
up: "[[Proxmox Virtual Environment]]"
title: Cloud-Init
topic: ProxmoxVE
description: Process to create a cloud-init template virtual machine
---
## Cloud Init
Cloud images are operating system templates and every instance starts out as an identical clone of every other instance. It is the user data that gives every cloud instance its personality and cloud-init is the tool that applies user data to your instances automatically.
[Cloud-init.io](https://cloud-init.io)
### Cloud Init Images
- [Debian](https://cloud.debian.org/images/cloud/)
- [Ubuntu](https://cloud-images.ubuntu.com/)
- [Alpine](https://www.alpinelinux.org/cloud/)
- [Alma](https://wiki.almalinux.org/cloud/)
- [Rocky](https://wiki.rockylinux.org/rocky/image/#not-recommended-avoid)
Other cloud-init images are, of course, available. In most cases, Debian or Ubuntu would be used.
## Proxmox Process
1. Download chosen distribution cloud image to Proxmox:
``` shell
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
```
2. Create a new VM
```shell
qm create <vmid> --memory 2048 --core 2 --name <vm-name> --net0 virtio,bridge=vmbr0
```
- vmid is the numerical ID of the VM. The network interface (vmbr0) will need to match the virtual interface you wish to use.

3. Import the downloaded image to `local` storage (or whichever storage you choose)
```shell
qm disk import <vmid> <downloaded-disk-image> local
```
4. Attach the new disk to the created VM as a SCSI drive on the SCSI controller (storage of your choice)
```shell
qm set <vmid> --scsihq virtio-scsi-pci --scsi0 local:vm-<vmid>-disk-0
```
5. Add cloud init drive (change `local` to the storage of choice)
```shell
qm set <vmid> --ide2 local:cloudinit
```
6. Make the cloud-init drive bootable and restrict BIOS to boot from disk only
```shell
qm set <vmid> --boot c --bootdisk scsi0
```
7. Add a serial console
```shell
qm set <vmid> --serial0 socket --vga serial0
```
8. Configure the cloud-init parameters:
	1. username
	2. password
	3. ssh key (strongly recommended)
	4. Upgrade packages
	5. IP Config (defaults to "NONE")
9. Create a template
```shell
qm template <vmid>
```
10. Once the template is complete, create a FULL CLONE of the template to create a new VM. Once the new VM is created, the various parameters (hardware) can be adjusted. Recommend increasing boot-disk size to 32/64/128GB