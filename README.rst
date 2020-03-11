===============
QEMU Cheatsheet
===============

Please make this your cheatsheet and suggest any change and/or addition.


Installing From Source
======================

.. code-block:: bash

  $ sudo apt-get install git-email libaio-dev libbluetooth-dev libbrlapi-dev libbz2-dev libcap-dev libcap-ng-dev libcurl4-gnutls-dev libgtk-3-dev libibverbs-dev libjpeg8-dev libncurses5-dev libnuma-dev librbd-dev librdmacm-dev libsasl2-dev libsdl1.2-dev libseccomp-dev libsnappy-dev libssh2-1-dev libvde-dev libvdeplug-dev  libxen-dev liblzo2-dev valgrind xfslibs-dev libnfs-dev libiscsi-dev

  $ cd ~/git; git clone git://git.qemu-project.org/qemu.git

  $ cd qemu; git checkout -b origin/stable-x.y

  $ ./configure --target-list=x86_64-softmmu

  $ make -j12


Create Ubuntu VM
================

.. code-block:: bash

  $ mkdir ~/qemu-images; cd ~/qemu-images

  $ wget http://releases.ubuntu.com/18.04.4/ubuntu-18.04.4-live-server-amd64.iso

  $ ~/git/qemu/qemu-img create ubuntu1804-qemu-unspoiled-installation.img 8G

  # To boot and install:
  $ sudo ~/git/qemu/x86_64-softmmu/qemu-system-x86_64 -m 2048 -enable-kvm -smp 2 -drive file=~/qemu-images/ubuntu1804-qemu-unspoiled-installation.img,format=raw -cdrom ~/qemu-images/ubuntu-18.04.4-live-server-amd64.iso -boot d

  # To run the VM to work inside it. This would be the first boot after
  # the installation:
  $ sudo ~/git/qemu/x86_64-softmmu/qemu-system-x86_64 -m 2048 -enable-kvm -smp 2 -drive file=~/qemu-images/ubuntu1804-qemu-unspoiled-installation.img,format=raw
  # Consider leave "unspoiled" that image and allways copy it before start
  # changing anything as suggesting in the next section.
  

General Configuration (Spoiling the Image)
==========================================

.. code-block:: bash

  # To backup VM image.
  $ cp ~/qemu-images/ubuntu1804-qemu-unspoiled-installation.img ~/qemu-images/ubuntu1804-qemu-spoiled-installation.img

  # To run the VM to work inside it.
  $ sudo ~/git/qemu/x86_64-softmmu/qemu-system-x86_64 -m 2048 -enable-kvm -smp 2 -drive file=~/qemu-images/ubuntu1804-qemu-spoiled-installation.img,format=raw

  # To be able to see more messages during booting time do inside the VM:
  $ sudo vim /etc/default/grub
  # set variable: GRUB_CMDLINE_LINUX="console=ttyS0" and then:
  $ sudo update-grub

  # Now you can poweroff the VM and start it again with the following command 
  # to avoid the UI Window while still having access to Kernel booting messages:
  $ sudo ~/git/qemu/x86_64-softmmu/qemu-system-x86_64 -m 2048 -enable-kvm -smp 2 -s -nographic -drive file=~/qemu-images/ubuntu1804-qemu-spoiled-installation.img,format=raw


Networking Tweaks
=================

User Networking (SLIRP) (Default)
---------------------------------

.. code-block:: bash

  # This is with QEMU user networking backend, no scripts are needed and it
  # should let you access the Internet. Ping (ICMP) will not work, though TCP
  # and UDP will. I use this configuration to update the rootfs using the
  # distro kernel. You can do "ssh -p 5555 user@localhost" to connect to the
  # VM because we are forwarding the host port 5555 to the guest port 22.
  $ sudo ~/git/qemu/x86_64-softmmu/qemu-system-x86_64 -m 2048 -enable-kvm -smp 2 -s -nographic -drive file=~/qemu-images/ubuntu1804-qemu.img,format=raw -nic user,hostfwd=tcp::5555-:22


Tap Networking (Best Performance)
---------------------------------

.. code-block:: bash

  # If you are looking to run any kind of network service or have your guest
  # participate in a network in any meaningful way, tap is usually the best
  # choice. Run the VM to change network configuration in the guest:
  $ sudo ~/git/qemu/x86_64-softmmu/qemu-system-x86_64 -m 2048 -enable-kvm -smp 2 -s -nographic -drive file=~/qemu-images/ubuntu1804-qemu-spoiled-installation.img,format=raw
  
  # Inside the VM  do:
  $ sudo vim /etc/netplan/50-cloud-init.yaml
  # and edit it taking as an example "qemu-scripts/50-cloud-init.yaml". Be
  # aware that the OS could be identifying the virtual NIC with a different
  # name, e.g. ens3, enp0s3, ...
  $ sudo netplan apply

  $ sudo poweroff

  # To boot with Tap networking:
  $ sudo ~/git/qemu/x86_64-softmmu/qemu-system-x86_64 -m 2048 -enable-kvm -smp 2 -s -nographic -drive file=~/qemu-images/ubuntu1804-qemu-spoiled-installation.img,format=raw -nic tap,script=FULL-PATH-TO/qemu-scripts/qemu-ifup-tap,downscript=no
  # Now from your host you will be able to treat the guest as another network
  # host in the network, e.g. "ssh 192.168.8.10" to connect the guest if that
  # was the IP address specified in the "/etc/netplan/50-cloud-init.yaml" guest
  # file. From outside your host, nobody will be able to access your guest and
  # your guest will have access to the Internet through NAT.

**TODO**

#. Write a "downscript" to clean up.
#. Improve network security.
#. Facilitate to run multiple VMs each with a different IP address.


Specifying Kernel Image
=======================

.. code-block:: bash

  $ sudo ~/git/qemu/x86_64-softmmu/qemu-system-x86_64 -kernel ~/git/kernels/linux/arch/x86/boot/bzImage -append 'root=/dev/sda2 rw console=ttyS0,115200n8 acpi=off nokaslr' -m 2048 -enable-kvm -smp 2 -s -nographic -drive file=~/qemu-images/ubuntu1804-qemu-spoiled-installation.img,format=raw -nic tap,script=FULL-PATH-TO/qemu-scripts/qemu-ifup-tap,downscript=no
  # With this command, the name of the network interface could change, e.g. form
  # "ens3" to "enp0s3". If so, update "/etc/netplan/50-cloud-init.yaml" in the
  # guest and run "sudo netplan apply".


References
==========
- https://wiki.debian.org/QEMU
- https://wiki.qemu.org/Hosts/Linux
- https://wiki.qemu.org/Documentation/Networking
- `KD0 - Preparing a Test Machine <https://medium.com/@aciliketcap/kd0-preparing-a-test-vm-54ed4299c23c>`_
- `KD1a - Creating Gentoo Userspace <https://medium.com/@aciliketcap/kd1-a-creating-gentoo-userspace-7d8f36f5268>`_
- `KD2a - Configuring and Compiling Kernel for QEMU VM <https://medium.com/@aciliketcap/kd2a-configuring-and-compiling-kernel-for-amd64-inside-qemu-b687a3178bed>`_
- `KD2a - Configuring and Compiling Kernel for QEMU VM <https://medium.com/@aciliketcap/kd2a-configuring-and-compiling-kernel-for-amd64-inside-qemu-b687a3178bed>`_
- `KD3a - Running Kernel with QEMU VM <https://medium.com/@aciliketcap/kd3-running-kernel-under-qemu-test-vm-11cc66b2b6e9>`_
- `KD4 - Compile and Run a Module <https://medium.com/@aciliketcap/kd4-compile-and-run-a-module-be7891cdb373>`_
- https://help.ubuntu.com/lts/serverguide/firewall.html
- https://www.linux-kvm.org/page/Networking
- https://qemu.weilnetz.de/doc/qemu-doc.html#Network-options
