# Linux Unattended Installation (LUI)

This fork of coreprocess/linux-unattended-installation is built to address a specific requirement when cloud building from scratch. When there is only unconfigured bare metal, there is no router, no beachhead, nothing, nada!  Fortunately, coreprocess provided the essentials to solve this problem by adding config files and additional boot content:  

 ![Bootstrapping a cloud](https://static.alta3.com/images/cloud-bootstrap.png)

## Rebuilding a cloud for sport
There is something *very* gratifying in creating the ability to build a working cloud from nothing but bare metal. No router, no beachhead, no compute nodes, nothing! Bootstrapping a cloud in this condition shoud be everyone's goal. Bootstrap order:
1. bring up a router, dns, dhcp entirely from preseed
2. bring up a beachhead server to cloud ansible playbooks and start building entirely from preseed.
3. bring up the compute nodes
4. Unleash ansible to configure everything.
5. Try to be completely restored in less that 20 mintutes total. 
6. WARNING: You will NEVER hit that goal without a well written bootstrap process

## The router
A minimal preseed will not work for this task. A router is born directly connected to the internet and must begin its life as a secure host. Since there is no other services, the router must be created autonimously from preseed. The issues are:
- Push a netplan to configure inside and outside interfaces
- Install DNSMasq along with the config, host-dns, host-dhcp, dhcp-options files which are all quite small at bootstrap.
- Set up iptables to handle basic port forwards and ip masquerade
- Set up a user (not root), RSA keys, and visudo to manage the newly born router.
- Build the router on a minimal ubuntu 20.04 OS
- Make sure all other services are turned off
- There may be more to be added to this list

## Building an SSH landing point (Beachhead)
Once the router is in place, a reliable landing point is needed as the base of operations. We call this point "beachhead"  This is where ansible will be run to do the heavy lifting necessary to configure the compute nodes. This server must also be built autonimously from preseed
- Centralized logging
- repository mirror
- Other essential services

All of this requires an unattended installation of a minimal setup of Linux, whereas *minimal* translates to the most lightweight setup - including an OpenSSH service and Python - which you can derive from the standard installer of a Linux distribution. The idea is, you will do all further deployment of your configurations and services with the help of Ansible or similar tools once you completed the minimal setup.

## Scrapping Ubuntu 16.04 LTS and 18.04 LTS

We are dropping support for 16.04 and 18.04

## Ubuntu 20.04 LTS

We will create three preseed types
- Router
- Beachhead
- Compute Node (coreprocess has this just about perfect)

### Features

* Fully automated installation procedure.
* Shutdown and reboot when finished. 
* Authentication based on SSH public key and NEVER on a password.
* Setup to use 100% of free disk space.
* Generates SSH server keys on first boot and not during setup stage. We consider this a feature since it enables you to use the installed image as a template for multiple machines.
* Print IPv4 and IPv6 address of the device on screen once booted.
* USB bootable hybrid ISO image.
* UEFI and BIOS mode supported.

### Prerequisites

#### Linux

Run `sudo apt-get install dos2unix p7zip-full cpio gzip genisoimage whois pwgen wget fakeroot isolinux xorriso` to install software tools required by the `build-iso.sh` script.

Run `sudo apt-get install qemu-utils qemu-kvm` in addition to install software tools required by the `build-disk.sh` script.


#### Docker - Removed from this branch

#### Build disk images - Removed from this branch.

### Usage

#### Build ISO images

You can run the `build-iso.sh` script as regular user. No root permissions required.

```sh
./ubuntu/<VERSION>/build-iso.sh <ssh-public-key-file> <target-iso-file>
```

All parameters are optional.

| Parameter | Description | Default Value |
| :--- | :--- | :--- |
| `<ssh-public-key-file>` | The ssh public key to be placed in authorized_keys | `$HOME/.ssh/id_rsa.pub` |
| `<target-iso-file>` | The path of the ISO image created by this script | `ubuntu-<VERSION>-netboot-amd64-unattended.iso` |

Boot the created ISO image on the target VM or physical machine. Be aware the setup will start within 10 seconds automatically and will reset the disk of the target device completely. The setup tries to eject the ISO/CD during its final stage. It usually works on physical machines, and it works on VirtualBox. It might not function in certain KVM environments in case the managing environment is not aware of the *eject event*. In that case, you have to detach the ISO image manually to prevent an unintended reinstall.

Power-on the machine and log into it as root using your ssh key. The ssh host key will be generated on first boot.



## Notes on debian preseed
Before you start building, consider a little practice to see what is going on, lets make a few assumptions:

Your USB key has a single FAT32 PARTITION and appears as: 
`/dev/sdb1`  

You remove USB WRITE PROTECTION as follows  
`sudo hdparm -r0 /dev/sdb1`  

You will create the router ISO like this:  
`./ubuntu/20.04/build-iso.sh /home/ubuntu/.ssh/id_rsa.pub  /home/ubuntu/router.iso`  

You will create the USB key using DD like this:  
`sudo dd bs=4M if=/home/ubuntu/router-v1.iso of=/dev/sdb1 conv=fdatasync status=progress`

Check your work, so mount the USB key  
`mkdir /usb`
`mkdir ~/router`
`mount /dev/sdb1 /usb`  
`cd /usb`
`cp initrd.gz ~/router`
`cd ~/router`
`gunzip initd.gz`
`cpio -i < initd`
Check the contents of the custom directory
Make sure the preseed file matches what you created the iso with in the first place from `./ubuntu/20.04/build-iso.sh /home/ubuntu/.ssh/id_rsa.pub  /home/ubuntu/router.iso`


