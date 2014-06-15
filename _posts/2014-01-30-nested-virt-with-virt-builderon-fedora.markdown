---
layout: post
title: "KVM-based nested virtualization setup with virt-builder on Fedora"
date: 2014-01-30 14:42:48 +0530
comments: true
categories: 
- Nested Virtualization 
- KVM
- virt-tools
---

Install packages and check KVM module configuration
---------------------------------------------------

Install minimal KVM-based virtualization and rawhide release packages:

      $ yum install libvirt-daemon-kvm                             \
      libvirt-daemon-config-network libvirt-daemon-config-nwfilter \
      virt-install libguestfs libguestfs-tools libguestfs-tools-c -y
    
      $ yum install fedora-release-rawhide -y


Update Kernel, QEMU, Libvirt, Libguestfs to latest from Rawhide:

      $ yum update kernel qemu-system-x86 libvirt-daemon-kvm \
      libguestfs virt-install --enablerepo=rawhide


Ensure `nested`, `enable_shadow_vmcs`, `ept` KVM kernel module
parameters are enabled on L0.

      $ cat /sys/module/kvm_intel/parameters/nested         \
        /sys/module/kvm_intel/parameters/enable_shadow_vmcs \
        /sys/module/kvm_intel/parameters/ept                \
        /sys/module/kvm_intel/parameters/enable_apicv       \
      Y
      Y
      Y
      N

If it doesn't reflect as above, add "options kvm-intel nested=y"
(without quotes) to `/etc/modprobe.d/dist.conf`, & reboot the host.


Create a new libvirt bridge
---------------------------

Create a new libvirt network (other than your default 198.162.x.x) file:

      $ cat testnet2.xml 
      <network>
        <name>testnet2</name>
        <uuid>d0e9964a-f91a-40c0-b769-a609aee41bf2</uuid>
        <forward mode='nat'>
          <nat>
            <port start='1024' end='65535'/>
          </nat>
        </forward>
        <bridge name='virbr1' stp='on' delay='0' />
        <mac address='52:54:00:60:f8:6e'/>
        <ip address='192.169.142.1' netmask='255.255.255.0'>
          <dhcp>
            <range start='192.169.142.2' end='192.169.142.254' />
          </dhcp>
        </ip>
      </network>


Define the above network, start it, ensure it is listed in Libvirt
active networks (& optionally list the bridge devices):

    $ virsh net-define testnet2.xml
    $ virsh net-start testnet2
    $ virsh net-autostart testnet2
    $ brctl show    

Setup L1 (guest hypervisor)
---------------------------

Use virt-builder to create a VM:

      $ LIBGUESTFS_BACKEND=direct
 
      $ virt-builder fedora-20 --format qcow2 --size 100G
      virt-builder fedora-20 --format qcow2 --size 100G
      [   1.0] Downloading: http://libguestfs.org/download/builder/fedora-20.xz
      #######################################################################  100.0%
      [ 131.0] Planning how to build this image
      [ 131.0] Uncompressing
      [ 139.0] Resizing (using virt-resize) to expand the disk to 100.0G
      [ 220.0] Opening the new disk
      [ 225.0] Setting a random seed
      [ 225.0] Setting random root password [did you mean to use --root-password?]
      Setting random password of root to N4KkQjZTgdfjjqJJ
      [ 225.0] Finishing off
      Output: fedora-20.qcow2
      Output size: 100.0G
      Output format: qcow2
      Total usable space: 97.7G
      Free space: 97.0G (99%)


Give execute permission to /home/test directory and set the correct
SELinux context and boot into it:

      $ chmod o+x /home/test/

      $ virt-install --name guest-hyp --ram 8192 --vcpus=4 \
        --disk path=/home/test/vmimages/fedora-20.qcow2,format=qcow2,cache=none --import

      $ virsh console guest-hyp
      $ init 6 # inside L1


Edit the L1's libvirt XML:

      $ virsh edit guest-hyp


And, add the fragment:

      <cpu mode='host-passthrough'/>

**NOTE: Ensure to boot L1 guest w/ libvirt's non-default network bridge (virbr1).**


Setup L2 (nested guest)
-----------------------

Install KVM virt packages (refer L1 setup), check KVM module
configuration.


Create L2 guest:

      $ virt-builder fedora-20 --format qcow2 --size 10G
      virt-builder: warning: cache /root/.cache/virt-builder: Unix.Unix_error(Unix.ENOENT, "mkdir", "/root/.cache/virt-builder")
      virt-builder: disabling the cache
      [   1.0] Downloading: http://libguestfs.org/download/builder/fedora-20.xz
      #######################################################################   99.9%
      [ 187.0] Planning how to build this image
      [ 187.0] Uncompressing
      [ 195.0] Resizing (using virt-resize) to expand the disk to 10.0G
      [ 267.0] Opening the new disk
      [ 274.0] Setting a random seed
      [ 274.0] Setting random root password [did you mean to use --root-password?]
      Setting random password of root to zuZgCD2WnR6kNw8Y
      [ 275.0] Finishing off
      Output: fedora-20.qcow2
      Output size: 10.0G
      Output format: qcow2
      Total usable space: 9.1G
      Free space: 8.4G (92%)


Import it to access serial console:

    $ virt-install --name nguest1 --ram 2048 --vcpus=2 \
    --disk
    path=/home/tuser1/vmimages/fedora-20.qcow2,format=qcow2,cache=none
    --import
