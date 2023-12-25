---
title: 'Create Virsh Ubuntu VM'
date: 2023-12-25T02:15:17+01:00
draft: false
tag:
    - virtualization
    - libvirt
---
## Download qcow image
Download qcow image from [Ubuntu repo](https://cloud-images.ubuntu.com)
## Create network xml
Examples can be found [here](https://libvirt.org/formatnetwork.html#example-configuration)
```xml
<network>
  <name>default</name>
  <bridge name="virbr0"/>
  <forward mode="nat"/>
  <ip address="192.168.122.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.122.2" end="192.168.122.254"/>
    </dhcp>
  </ip>
</network>
```
## Create network
`sudo virsh net-define /path/to/network.xml`
`sudo virsh net-start <network-name>`
## Create simple domain xml
Examples can be found [here](https://libvirt.org/drvqemu.html#example-domain-xml-config)
To boot from disk disk type must be defined (driver tag).
```xml
<domain type='qemu'>
  <name>nginx-vm</name>
  <memory>419200</memory>
  <vcpu>2</vcpu>
  <os>
    <type arch='x86_64' machine='pc'>hvm</type>
    <boot dev="hd" />
  </os>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type='file' device='disk'>
      <source file='/path/to/image.qcow'/>
      <target dev='hda'/>
      <driver name='qemu' type='qcow2' cache='none'/>
    </disk>
    <interface type='network'>
      <source network='default'/>
    </interface>
    <serial type='pty'>
      <target port='0'/>
    </serial>
    <console type='pty'>
      <target type='serial' port='0'/>
    </console>
    <graphics type='vnc' port='-1'/>
  </devices>
</domain>
```

## Trobleshooting
* If you are not able to connect to console (`virsh console <vm-name>`). You can try to use vncviewer to connect to vm. At `/var/log/libvirt/qemu/<vm-name>.log` you can find vnc connection details
