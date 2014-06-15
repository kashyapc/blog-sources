---
layout: post
title: "Build libguestfs inside a `systemd-nspawn` container"
date: 2014-01-30 14:43:48 +0530
comments: true
categories:
- libguestfs
- systemd
- containers
---

Prerequisite
------------

Because of an audit subsystem incompatibility [bug], we need to turn off
auditing by booting the host with `audit=0` on Kernel command line. (As
of writing this, there's an [upstream patch review] in progress to fix
this.)

Install a minimal Fedora distribution
-------------------------------------

On the host, specify an installroot (/srv) and install a minimal Fedora
20 distribution:

    $ yum -y --releasever=20 --nogpg        \
      --installroot=/srv/testcontainer      \
      --disablerepo='*' --enablerepo=fedora \
      install systemd passwd yum            \
      fedora-release vim-minimal

Invoke `systemd-nspawn` 
----------------------

Boot into the container, set the password:

    $ systemd-nspawn -D /srv/testcontainer
    [. . .]
    -bash-4.2# passwd

Start the container w/ systemd:

    $ systemd-nspawn -bD /srv/testcontainer
    [. . .]
    -bash-4.2#

Introspect the container
------------------------

While the container is running, get the runtime status information of it
using `machinectl`:

    $ machinectl status testcontainer
    testcontainer
               Since: Wed 2014-01-29 21:16:58 EST; 13h ago
              Leader: 10844 (systemd)
             Service: nspawn; class container
                Root: /srv/testcontainer
                Unit: machine-testcontainer.scope
                      ├─10844 /usr/lib/systemd/systemd
                      ├─system.slice
                      │ ├─lvm2-lvmetad.service
                      │ │ └─10879 /usr/sbin/lvmetad
                      │ ├─dbus.service
                      │ │ └─10895 /bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
                      │ ├─systemd-logind.service
                      │ │ └─10894 /usr/lib/systemd/systemd-logind
                      │ ├─rpcbind.service
                      │ │ └─10897 /sbin/rpcbind -w
                      │ ├─libvirtd.service
                      │ │ └─10896 /usr/sbin/libvirtd
                      │ ├─crond.service
                      │ │ └─10904 /usr/sbin/crond -n
                      │ └─systemd-journald.service
                      │   └─10855 /usr/lib/systemd/systemd-journald
                      └─user.slice
                        └─user-0.slice
                          ├─session-8.scope
                          │ ├─10905 login -- root
                          │ └─11046 -bash
                          └─user@0.service
                            ├─11033 /usr/lib/systemd/systemd --user
                            └─11045 (sd-pam)


Some more container introspection:

    $ systemd-cgls --machine=testcontainer

    $ systemd-cgtop

    $ journalctl -D /srv/testcontainer/var/log/journal

Building Libguestfs
-------------------

Inside the minimal Fedora 20 container, install libguestfs dependencies,
clone the libguestfs git repository:

    -bash-4.2# yum-builddep libguestfs -y

    -bash-4.2# git clone git://github.com/libguestfs/libguestfs.git

Build and invoke the libguestfs test suite in the container:

    -bash-4.2# cd libguestfs

    -bash-4.2# ./autogen.sh && time make 2>&1 \
                | tee /tmp/libguestfs-compile.log

     -bash-4.2# time make -k check \
     LIBGUESTFS_DEBUG=1 LIBGUESTFS_TRACE=1 2>&1 \
     | tee /tmp/libguestfs-test.log


Notes
-----

- If you need to build a container without networking (once all the
  relevant dependencies are installed/cloned over the network), you can
  invoke the container without devices:

        $ systemd-nspawn -bD /srv/testcontainer --private-network
        [. . .]
        -bash-4.2# 

- References: [systemd Container Interface] document; [virtualized testing] systemd.

[bug]:https://bugzilla.redhat.com/show_bug.cgi?id=966807
[upstream patch review]:https://www.redhat.com/archives/linux-audit/2013-May/msg00065.html
[systemd Container Interface]:http://www.freedesktop.org/wiki/Software/systemd/ContainerInterface/
[virtualized testing]:http://www.freedesktop.org/wiki/Software/systemd/VirtualizedTesting/
