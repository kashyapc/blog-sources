---
layout: post
title: "PCI passthrough of an audio card on a Intel machine (with IOMMU)"
date: 2014-03-22 14:42:48 +0530
comments: true
categories: 
- Virtualization
- KVM
- QEMU
- Fedora
---

I remember from one of the [hallway conversations] with Luke Macken at
Flock 2013, in Charleston, he was trying out PCI passthrough of an Audio
device that didn't work for him. Last night, I tried it out on a Fedora
20 machine, and it worked. Here's how.


Check/enable Intel IOMMU on host
--------------------------------


Check if VT-d (Intel IOMMU) is enabled:

    $ dmesg -H -T | grep -i iommu


If it's not present, enable it by adding `intel_iommu=on` to
/etc/grub2.cfg, reboot, and run `dmesg` again:

    $ dmesg -H -T | grep -i iommu
    [Sat Mar 22 00:41:30 2014] Command line: BOOT_IMAGE=/vmlinuz-3.13.4-200.fc20.x86_64 root=/dev/mapper/fedora_dhcp193--46-root ro rd.lvm.lv=fedora_dhcp193-46/swap rd.md=0 rd.dm=0 rd.lvm.lv=fedora_dhcp193-46/root rd.luks=0 vconsole.keymap=us rhgb quiet LANG=en_US.UTF-8 intel_iommu=on
    [Sat Mar 22 00:41:30 2014] Kernel command line: BOOT_IMAGE=/vmlinuz-3.13.4-200.fc20.x86_64 root=/dev/mapper/fedora_dhcp193--46-root ro rd.lvm.lv=fedora_dhcp193-46/swap rd.md=0 rd.dm=0 rd.lvm.lv=fedora_dhcp193-46/root rd.luks=0 vconsole.keymap=us rhgb quiet LANG=en_US.UTF-8 intel_iommu=on
    [Sat Mar 22 00:41:30 2014] Intel-IOMMU: enabled
    [Sat Mar 22 00:41:30 2014] dmar: IOMMU 0: reg_base_addr fed90000 ver 1:0 cap c0000020e60262 ecap f0101a
    [Sat Mar 22 00:41:30 2014] dmar: IOMMU 1: reg_base_addr fed91000 ver 1:0 cap c9008020660262 ecap f0105a
    [Sat Mar 22 00:41:30 2014] IOAPIC id 2 under DRHD base  0xfed91000 IOMMU 1
    [Sat Mar 22 00:41:30 2014] IOMMU 0 0xfed90000: using Queued invalidation
    [Sat Mar 22 00:41:30 2014] IOMMU 1 0xfed91000: using Queued invalidation
    [Sat Mar 22 00:41:30 2014] IOMMU: Setting RMRR:
    [Sat Mar 22 00:41:30 2014] IOMMU: Setting identity map for device 0000:00:02.0 [0xdb800000 - 0xdf9fffff]
    [Sat Mar 22 00:41:30 2014] IOMMU: Setting identity map for device 0000:00:1d.0 [0xdacd5000 - 0xdacebfff]
    [Sat Mar 22 00:41:30 2014] IOMMU: Setting identity map for device 0000:00:1a.0 [0xdacd5000 - 0xdacebfff]
    [Sat Mar 22 00:41:30 2014] IOMMU: Prepare 0-16MiB unity mapping for LPC
    [Sat Mar 22 00:41:30 2014] IOMMU: Setting identity map for device 0000:00:1f.0 [0x0 - 0xffffff]


Add the Audio PCI device via `virsh`
------------------------------------

Identify the sound card on the host:

    $ lspci | grep -i audio
    00:1b.0 Audio device: Intel Corporation 6 Series/C200 Series Chipset Family High Definition Audio Controller (rev 04)


Enumerate the PCI vendor and device code for the above Audio device:

    $ lspci -n | grep 00:1b.0
    00:1b.0 0403: 8086:1c20 (rev 04)


Identify the string the PCI node device:

    $ virsh nodedev-list --tree | grep -i 1b
    +- pci_0000_00_1b_0


Dump the XML representation of the PCI node device:

    $ virsh nodedev-dumpxml pci_0000_00_1b_0
    <device>
      <name>pci_0000_00_1b_0</name>
      <path>/sys/devices/pci0000:00/0000:00:1b.0</path>
      <parent>computer</parent>
      <driver>
        <name>snd_hda_intel</name>
      </driver>
      <capability type='pci'>
        <domain>0</domain>
        <bus>0</bus>
        <slot>27</slot>
        <function>0</function>
        <product id='0x1c20'>6 Series/C200 Series Chipset Family High Definition Audio Controller</product>
        <vendor id='0x8086'>Intel Corporation</vendor>
        <iommuGroup number='5'>
          <address domain='0x0000' bus='0x00' slot='0x1b' function='0x0'/>
        </iommuGroup>
      </capability>
    </device>


Detach the above PCI node device, so it can be safely used by guests via
passthrough:

    $ virsh nodedev-detach pci_0000_00_1b_0
    Device pci_0000_00_1b_0 detached


Convert slot and function values to hexadecimal values (from decimal)
to get the PCI bus addresses. Append "0x" to the beginning of the output
to tell the computer that the value is a hexadecimal number. 

From the above XML representation, the bus, slot and function values
are: bus=0, slot=27, function=0

    $ printf %x 0
    0
    $ $ printf %x 27
    1b
    $ printf %x 0
    0

The values to use in the guest XML configuration:

    bus='0x00'
    slot='0x1b'
    function='0x00'


Edit the guest XML, and add a <device> attribute to attach the Audio
card to the guest:

    $ virsh edit f20vm
    [. . .]


Add the below XML xnippet and save the configuration:

    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
          <address domain='0x0000' bus='0x00' slot='0x1a' function='0x7'/>
      </source>
    </hostdev>


Once saved, dump the XML representation to ensure it reflects the new
reality (i.e. the slot (0x1b) of the sound card should be visible):  

    $ virsh dumpxml ostack-controller | grep 1b -B2 -A3
        <hostdev mode='subsystem' type='pci' managed='yes'>
          <source>
            <address domain='0x0000' bus='0x00' slot='0x1b' function='0x0'/>
          </source>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
        </hostdev>


Ensure the device is under the PCI stub (since we already detached the
device from the host): 

    $ readlink /sys/bus/pci/devices/0000\:00\:1b.0/driver
    ../../../bus/pci/drivers/pci-stub


Detach the device:

    $ virsh nodedev-detach pci_0000_00_1b_0
    Device pci_0000_00_1b_0 detached


Ensure VFIO is being used:

    $ lsmod | grep -i vfio
    vfio_iommu_type1       17636  0 
    vfio                   20160  1 vfio_iommu_type1


Temporarily, set SELinux to permissive:

    $ setenforce 0
    $ virsh start foo


Inside the guest, run `lspci` again and grep for the Audio card:

    $ lspci | grep -i Audio
    00:05.0 Audio device: Intel Corporation 6 Series/C200 Series Chipset Family High Definition Audio Controller (rev 04)


To reattach device back to the host
-----------------------------------

Shutdown the guest:

    $ virsh shutdown foo

Re-attach the device node again:

    $ virsh nodedev-reattach pci_0000_00_1b_0
    Device pci_0000_00_1b_0 re-attached


NOTES
-----

Thanks to Alex Williamson.

 - The PCI slot inside the guest won't be at the same address as the
   host unless you configure it to be.

 - Sanity test: For audio where you can't physically hear the output,
   pull up something like pandora and watch interrupts in the host to
   see if the device is continuing to generate interrupts

        $ watch -d grep vfio /proc/interrupts




[hallway conversations]:http://kashyapc.com/2013/08/17/flock-2013-retrospective/
