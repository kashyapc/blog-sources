---
layout: post
title: "Debugging KVM-based nested virtualization problems"
date: 2014-01-30 14:42:48 +0530
comments: true
categories: 
- Nested Virtualization 
- KVM
- libguestfs
- Debugging
---

Install utilities for debugging
------------------------------

    $ yum install libguestfs qemu-sanity-check --enablerepo=rawhide -y


Debugging methods
-----------------

  1. Run the `libguesfs-test-tool` in L1 (which has KVM extensions
    exposed) with 'direct' backend:

        $ export LIBGUESTFS_BACKEND=direct
        $ guestfish get_backend
        direct
        $ export LIBGUESTFS_DEBUG=1
        $ export LIBGUESTFS_TRACE=1
        $ libguestfs-test-tool |& stderr-direct-backend.txt

  2. If L2 is stuck with KVM, try TCG (Software emulation):

        $ export LIBGUESTFS_BACKEND_SETTINGS=force_tcg
        $ export LIBGUESTFS_BACKEND=direct
        $ guestfish get_backend
        direct
        $ libguestfs-test-tool |& tee stderr-direct-backend.txt

  3. Also can run the QEMU sanity check `qemu-sanity-check`

        $ qemu-sanity-check |& tee stderr-qemu-sanity-check.txt

  4. If the above doesn't work, try attaching GDB to QEMU:

        https://github.com/libguestfs/libguestfs/blob/master/src/launch-direct.c#L404

     Need to re-build libguestfs with `gdb` debugging enabled
     in /src/launch-direct.c:

        $ git clone git://github.com/libguestfs/libguestfs.git

     Activate gdb debugging code by turning on its conditional
     compilation directive: s/#if 0/#if 1

        $ vi src/launch.c
        [. . .] # Edit, save it

    Ensure the pre-processor directive is turned on:

        $ grep "\-S" src/launch-direct.c -A4 -B2
           */
        #if 1
          ADD_CMDLINE ("-S");
          ADD_CMDLINE ("-s");
          warning (g, "qemu debugging is enabled, connect gdb to tcp::1234 to begin");
        #endif

    Compile:

        $ ./autogen.sh
        $ make -j4
        [. . .] # Compile, address anything that comes up
        $ echo $?
        0

    Install GDB, etc

        # NOTE -- The installed size will be 1.5 G
        $ yum install --enablerepo=fedora-debuginfo kernel-debuginfo

    Then run the appliance:

        $ ./run libguestfs-test-tool  # This should invoke it

    From a different terminal, invoke `gdb`

        (gdb) symbol-file
        /usr/lib/debug/lib/modules/3.14.0-0.rc2.git4.1.fc21.x86_64/vmlinux
        Reading symbols from
        /usr/lib/debug/lib/modules/3.14.0-0.rc2.git4.1.fc21.x86_64/vmlinux...done.
        (gdb) target remote tcp::1234
        Remote debugging using tcp::1234
        0x0000fff0 in cpu_lock_stats ()
        (gdb) cont
        Continuing.

        # [Step through gdb] 

  5. Further tips:

      - A tip from [here] indicates to look in kernel log_buf.
          
      - To capture guest %rip and other registers via gdb, take a look
        at [this].
        
      - Drop back to older CPU models on both L1 and L2

[this]:http://lists.gnu.org/archive/html/qemu-devel/2011-09/msg03772.html
[here]: http://www.wiki.xilinx.com/QEMU-Linux+Kernel+logbuf+Extraction

