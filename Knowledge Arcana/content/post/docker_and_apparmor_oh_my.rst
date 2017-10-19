+++
title = "Docker and the case of the denied libraries"
tags = [ "docker" ]
date = "2017-10-18"
categories = [
  "sysadmin"
]
slug = "docker_shlibs"
+++

Today, when spawning a container using our (Cloudflare) dev env, I was presented with the following unexpected error:

..

   /usr/sbin/syslog-ng: error while loading shared libraries: libsyslog-ng-3.6.so.0: cannot open shared object file: Permission denied

For background we use salt SaltStack for config management and orchestration of bare metal, we then shoehorn this into a debian container for testing the majority of config changes. So container starts, begins highstating, and bam, syslog-ng can't start, WTF.

Check out a random commit from the previous day where I was able to work on this (unrelated) change before. No luck. Fine.

In the container lets try running syslog-ng by hand:

.. code-block:: sh

   root@dev:/# /usr/sbin/syslog-ng -F --fd-limit 100000 --worker-threads 4
   /usr/sbin/syslog-ng: error while loading shared libraries: libsyslog-ng-3.6.so.0: cannot open shared object file: Permission denied
   root@dev:/# 

What the @#$%* is going on?

.. code-block:: sh

   root@dev:/lib# strace -f /usr/sbin/syslog-ng -F --fd-limit 100000 --worker-threads 4 2>&1 | grep -e EACCES -e write
   open("/usr/lib/syslog-ng/libsyslog-ng-3.6.so.0", O_RDONLY|O_CLOEXEC) = -1 EACCES (Permission denied)
   stat("/usr/lib/syslog-ng", 0x7ffd1848faf0) = -1 EACCES (Permission denied)
   open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = -1 EACCES (Permission denied)
   stat("/lib/x86_64-linux-gnu", 0x7ffd1848faf0) = -1 EACCES (Permission denied)
   stat("/usr/lib/x86_64-linux-gnu", 0x7ffd1848faf0) = -1 EACCES (Permission denied)
   stat("/lib", 0x7ffd1848faf0)            = -1 EACCES (Permission denied)
   stat("/usr/lib", 0x7ffd1848faf0)        = -1 EACCES (Permission denied)
   writev(2, [{"/usr/sbin/syslog-ng", 19}, {": ", 2}, {"error while loading shared libra"..., 36}, {": ", 2}, {"libsyslog-ng-3.6.so.0", 21}, {": ", 2}, {"cannot open shared object file", 30}, {": ", 2}, {"Permission denied", 17}, {"\n", 1}], 10/usr/sbin/syslog-ng: error while loading shared libraries: libsyslog-ng-3.6.so.0: cannot open shared object file: Permission denied
   root@dev:/lib#
   root@dev:/lib# stat /usr/lib/syslog-ng/libsyslog-ng-3.6.so.0
     File: /usr/lib/syslog-ng/libsyslog-ng-3.6.so.0 -> libsyslog-ng-3.6.so.0.0.0
     Size: 25              Blocks: 0          IO Block: 4096   symbolic link
   Device: fd01h/64769d    Inode: 12195470    Links: 1
   Access: (0777/lrwxrwxrwx)  Uid: (    0/    root)   Gid: (    0/    root)
   Access: 2017-10-18 16:16:29.872617408 -0700
   Modify: 2017-02-10 01:25:38.000000000 -0800
   Change: 2017-06-14 16:08:56.317565396 -0700
    Birth: -
   root@dev:/lib#
   root@dev:/lib# stat /usr/lib/syslog-ng/libsyslog-ng-3.6.so.0.0.0
     File: /usr/lib/syslog-ng/libsyslog-ng-3.8.so.0.0.0
     Size: 655656          Blocks: 1288       IO Block: 4096   regular file
   Device: fd01h/64769d    Inode: 12193093    Links: 1
   Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
   Access: 2017-10-18 16:16:29.876617408 -0700
   Modify: 2017-02-10 01:25:38.000000000 -0800
   Change: 2017-06-14 16:08:56.313565543 -0700
    Birth: -

Not only can it not read the so file, it can't read any of the directories leading to it, yet permissions are right... right?
(This should have been my hint at a wider problem)

.. code-block:: sh

   root@dev:/lib# lsattr /usr/lib/syslog-ng/libsyslog-ng-3.6.so.0.0.0
   --------------e---- /usr/lib/syslog-ng/libsyslog-ng-3.6.so.0.0.0

   root@dev:/# getfacl usr/lib/syslog-ng/libsyslog-ng-3.6.so.0.0.0
   # file: usr/lib/syslog-ng/libsyslog-ng-3.6.so.0.0.0
   # owner: root
   # group: root
   user::rw-
   group::r--
   other::r--

It was about this time I noticed ntpd was failing to run in a similar way. One of my coworkers half jokingly asked if SELinux was turned on somehow, I said no... But I do run Ubuntu with AppArmor... no it can't be, I haven't changed it in ages.

.. code-block:: sh

   [vaelen:~]$ sudo grep DENIED /var/log/audit/audit.log | tail -n 1
   type=AVC msg=audit(1508369848.988:2773): apparmor="DENIED" operation="getattr" info="Failed name lookup - disconnected path" error=-13 profile="/usr/sbin/ntpd" name="var/lib/docker/aufs/diff/9604729a122cb5b8d2bbad3c121892ca2a589c4e6684a2de5f110949e2479b0f/lib/x86_64-linux-gnu" pid=1927 comm="ntpd" requested_mask="r" denied_mask="r" fsuid=0 ouid=0FSUID="root" OUID="root"

Smoking gun, but why? As a quick test I switched the syslog-ng apparmor profiles into complain mode and lo and behold, it was working. But that's not ok to disable a security mechanism if possible. A quick look at apt history and the profiles showed they hadn't changed in some time, something else must have.

Look back at the "name=" field, it has the full path to the file -on the host- the apparmor profile is checking paths to files accessed and nowhere has it allowed such a path, but its in a container, shouldn't the path be from the container's perspective?
After 1 second of googling, I came across https://github.com/moby/moby/issues/2800, aufs... Which I had changed out for overlay2 long ago was switched back on, setting it back and restarting docker, voila!

Now if only I knew how it had changed back.
