# Linux Unattended Installation (LUI)

This brach of coreprocess/linux-unattended-installation addresses a specific requirement when cloud building from scratch. When there is only unconfigured bare metal, which means; no router; no DHCP; no DNS; no ansible beachhead; and only a layer two switch then a lot of repetitive, skillful work is required to install the infrastructure. For small clouds, we need a fast, easy, and reliable way to build from sratch. Also, the solution must use existing low tech, adding no new or complex hardware and software. I other words, the budget eeds to be a $10 USB key for each server and that is all. Fortunately, coreprocess provided the essentials to solve this problem. This branch build on that, adding additional boot content for specific hosts, trying to honor the idea of simplicity,  but a willingness to trade some simplicty for speed of deployment in the case of router and beachhead building.   

 ![Bootstrapping a cloud](https://static.alta3.com/images/cloud-bootstrap.png)

## Rebuilding the cloud from bare metal
The process to rebuild a small cloud (less that 40 nodes) from bare metal should be fast, reliable, and easy. Most solutions require complex infrastructure to already be in place to build a cloud. Let's take a different route. Lets assume that NOTHING is in place.  We think a bootstrap USB drive in every server should be all that is necessary to build a small cloud and then let ansible do its work. Rebuilding should be as easy as booting the USB and let it do its work. This means we need a soluition that will work when there is no router, no DHCP, no beachhead, no compute nodes, nothing! Bootstrapping a cloud in this condition shoud be everyone's goal. Bootstrap order:
1. bring up a router, dns, dhcp entirely from preseed
2. bring up a beachhead server to cloud ansible playbooks and start building entirely from preseed.
3. bring up the compute nodes
4. Unleash ansible to configure everything.
5. Try to be completely restored in less that 20 mintutes total. 
6. WARNING: You will NEVER hit that goal without a well written bootstrap process

## The router
We want to build a router from Ubuntu, just like all our other hosts. Therefore, we can easily manage it with ansible as it appears as another host, stripped down to bare essentuials. Unfortunately, a minimal preseed will not work for this task. A router is born directly connected to the internet and must begin its life as a secure host with a public address which is not supplied by DHCP. A router also has at least two interfaces, but preseed can configure only one interface unless a netplan is pushed as a late command, which works well but this requires a custom preseed for our routers. Since there is no other services, the router must be created autonimously from preseed. The issues are:
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

## Scrapped in this branch:
- Ubuntu 16.04 LTS
- Ubuntu 18.04 LTS
- MAC support which requires a isohdpfx.bin binary with no MD5 checksum verification. 
- Docker 
- Build disk images 
- The long name "linux-unattended-installation" changed to "lui" for shorter command line instructions.

## Ubuntu 20.04 LTS

We will create three preseed types
- Router
- Beachhead
- Compute Node (some minor changes to coreprocess based build-iso script)

### Features

* Fully automated preseed for router/DNS/DHCP
* Fully automated preseed for ansible beachhead
* Fully automated preseed for compute nodes.
* Reboot when finished (rather than shutdown which requires manual start).
* Authentication based on SSH public key and NEVER on a password.
* Setup to use 100% of free disk space.
* Generates SSH server keys on first boot and not during setup stage. (Great job coreproccess!).
* Print IPv4 and IPv6 address of the device on screen once booted.
* Print "SSH ACCESS ONLY" on the console login.
* USB bootable hybrid ISO image.
* UEFI and BIOS mode supported.

### Install Dependancies

#### Ubuntu 20.04 ISO builder"

1. You MUST run an ubuntu 20.04 server to run our version of the `build-iso.sh` script which in turn makes your USB keys. An old laptop running ubuntu 20.04 is a great choice as your "ISO MAKER".  
    > We are using a 20.04 system to obtain `isohdpfx.bin` from `/usr/lib/ISOLINUX/isohdpfx.bin` because it was easier than going though the hassle of MD5 checkum verification, plus we wanted a simple, dedicated, secure server doing this work anyhow.

2. Install the following software on your "Ubuntu 20.04 ISO builder server".   

    `sudo apt-get install dos2unix p7zip-full cpio gzip genisoimage whois pwgen wget fakeroot isolinux xorriso` to install software tools required by the `build-iso.sh` script.

3. Clone the repository.

    `git clone https://github.com/alta3/lui.git`

4. CD into this repo

    `cd lui`

5. Change branch to this branch

    `git checkout cloud-bootstrap`

6. The `tree` command will show you something like this:

    ```
    ├── LICENSE
    ├── readme.md
    └── ubuntu
        ├── 20.04
        │   ├── build-iso.sh
        │   └── custom
        │       ├── boot-menu.patch
        │       ├── custom
        │       ├── preseed.cfg
        │       └── ssh-host-keygen.service
        ├── 20.04-beachhead
        │   ├── build-iso.sh
        │   └── custom
        │       ├── boot-menu.patch
        │       ├── custom
        │       ├── preseed.cfg
        │       └── ssh-host-keygen.service
        ├── 20.04-compute
        │   ├── build-iso.sh
        │   └── custom
        │       ├── boot-menu.patch
        │       ├── custom
        │       ├── preseed.cfg
        │       └── ssh-host-keygen.service
        └── 20.04-router
            ├── build-iso.sh
            └── custom
                ├── boot-menu.patch
                ├── custom
                ├── preseed.cfg
                └── ssh-host-keygen.service               
        ```

7. Edit the files in the custom directory for your 20.04-router, 20.04-compute, and 20.04-beachhead servers as approprate. All j2 files require ansible to process them before you can use them. 

8. Do you have an RSA key generated? YES or NO???

    `cat ~/.ssh/id_rsa.pub`

    > YES (GOOD)
    ```
        ssh-rsa AAAAB3NzaC1yc2EAAAAD - blah - blah - blah JyqrGaRLCC
    ```
    
    > NO (WORK NEEDED)
    ```
    No such file or directory`
    ```

9. If no, then create one as follows:

    `ssh-keygen` and accept the defaults.

10. Just for fun, run the `build-iso.sh` to create the most basic bootstrap usb. This one will work with no modication so that you can see how the software works.   

    `./ubuntu/20.04/build-iso.sh  ~/.ssh/.ssh/id_rsa.pub ~/my-first-iso.iso`  
    
    >It is not recommended, but you can do the same thing with no paramters, like this:

    `./ubuntu/20.04/build-iso.sh`
    
    > The software uses ~/.ssh/.ssh/id_rsa.pub by default and your iso will be called `ubuntu-20.04-netboot-amd64-unattended.iso`  You can see that all parameters are optional.

    | Parameter | Description | Default Value |
    | :--- | :--- | :--- |
    | `<ssh-public-key-file>` | The ssh public key to be placed in authorized_keys | `$HOME/.ssh/id_rsa.pub` |
    | `<target-iso-file>` | The path of the ISO image created by this script | `ubuntu-<VERSION>-netboot-amd64-unattended.iso` |

12. Now that you have an iso file, it is time to burn it on a USB key. Use `lsblk` to verify which **DEVICE NAME** your USB key is using. Most likely it will be`/dev/sdb1`, but in my case it was:

    ```
    lsblk
    
    NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
    sda                         8:0    0 698.7G  0 disk
    ├─sda1                      8:1    0   512M  0 part /boot/efi
    ├─sda2                      8:2    0     1G  0 part /boot
    └─sda3                      8:3    0 697.1G  0 part
      └─ubuntu--vg-ubuntu--lv 253:0    0   200G  0 lvm  /
    sdb                         8:32   1  29.9G  0 disk 
    └─sdb1                      8:33   1  29.9G  0 part  <------------ IN my case, DEVICE-NAME = /dev/sdb1 (yours may be different) 
    ```
    
13. Remove write protection from the USB drive. In my case this would be MY COMMAND, yours will depend on the **DEVICE-NAME** assignment

    If /dev/sdb1, then:
    `sudo hdparm -r0 /dev/sdb1`   <-- This is an EXAMPLE, use YOUR DEVICE-NAME, not this!!!
    
14. Use dd to write your iso to the USB. Make sure to substitute `<DEVICE-NAME>` with your USB device's name

    `sudo dd bs=4M if=/home/ubuntu/router-v1.iso of=/dev/DEVICE-NAME> conv=fdatasync status=progress`
    
15. Verification steps (for rocket scientists only)

    >Your USB key has a single FAT32 PARTITION which is most likley `/dev/sdb1` so I will use that as an example. You should mount your new ISO and verify what just happened as follows:  
    `sudo mkdir /usb`  
    `mkdir -p  ~/iso-stuff`  
    `sudo mount /dev/sdb1 /usb`  *ignore the read-only warning*  
    `cd /usb`  
    `sudo cp initrd.gz ~/iso-stuff`  
    `cd ~/iso-stuff/`  
    `gunzip initrd.gz` (Unzip the gzip file)  
    `ls` and notice that only `initrd` is present. That is because initrd is a CPIO archivie file (linux kernel can boot from CPIO archive, not tar, hence, CPIO archive is used)  
    `cpio -i < initrd` - This will unarchive the initrd file.  
    `ll` will reveal a posix file system!  
    Now look around. This is the filesystem that will boot in ramdisk, permitting unfettered access to the harddrives. Hopefully it should all make sense now, (at least it did for me.)  
    `umount /dev/sdb1`  and continue!  

16. Find a test machine that is OK to be COMPLETELY rebuilt. Nothing will remain on this target machine. (You have been warned!)

17. Set the TEST MACHINE BIOS to boot-select the hard drive as primary, USB as secondary, or the machine will potentially reboot from the USB reinstalling the OS over and over again. You certianly don't want this.

18. Plug your USB key into the test machine and boot select the USB key. Within 10 seconds the test machine disk will be reset completely. No turning back now, which is what you want, right?  

19. Your machine will complete the installtion in about 12 minutes (with 1 Gbps internet access and SSD drive in the test machine). When complete, setup ejects the ISO/CD.

20. When the machine boots, it will display the IPV4 and IPV6 addresses on the console.

21. SSH into your new machine as ubuntu@IPV4 address

22. FYI: The ssh host key of your new machine will be generated on first boot.

23. A neat trick is the dd the master boot record of the test machine SSD, then reboot the test machine. Since you clobbered the SSD of the test machine, the USB key will be selected as secondary and re-install. This time you can time how long it takes from reboot to OS reinstalled.

"Danke für ein tolles Repo" coreprocess!  (Ein Amerikaner bekommt nie die Chance, mit Deutsch zu üben) 



-----
## The hardest working line of code explained:
This repo uses xorriso (pronounced "chorizo")
Read this: https://www.gnu.org/software/xorriso/

In the script, you will see this line: `"$BIN_XORRISO" -as mkisofs -r -V "ubuntu_1804_netboot_unattended" -J -b isolinux.bin -c boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -input-charset utf-8 -isohybrid-mbr "$SCRIPT_DIR/custom/isohdpfx.bin" -eltorito-alt-boot -e boot/grub/efi.img -no-emul-boot -isohybrid-gpt-basdat -o "$TARGET_ISO" ./`

Understanding this line involves reading this:  https://wiki.osdev.org/Mkisofs
Each parameter is explained in the above URL, but I will break them out here as I want a clear reference before I start making changes.

| Flag | Argument  | Description |
| :--- | :--- | :--- |
|"$BIN_XORRISO" | |
|-as |mkisofs| mkisofs = Macintosh ISO File System|
|-r |  |  Create a normal linux filename with access permissions to make all files readable by everybody. |
|-V |"ubuntu_1804_netboot_unattended"| ISO 9660 Volume ID  Hmmm, why is this not "ubuntu_2004" ???|
|-J | | enables MS-Windows UCS-2 names via Joliet extension|
|-b | isolinux.bin |El Torito boot image for PC-BIOS|
|-c | boot.cat | El Torito Boot Catalog|
|-no-emul-boot| | Choose emulation NOT floppy emulation, rather ISOLINUX and GRUB2 |
|-boot-load-size| 4 | How many blocks of the boot image are to be loaded by the BIOS |
|-boot-info-table |  | Needed for boot images of ISOLINUX and GRUB2.|
|-input-charset|  utf-8 | Input charset that defines the characters used in local file names |
|-isohybrid-mbr | "$SCRIPT_DIR/custom/isohdpfx.bin" | SYSLINUX/ISOLINUX MBR template for El Torito boot image for BIOS |     
|-eltorito-alt-boot |   |  El Torito boot options will apply to the next boot image given by -b or -e|
|-e boot/grub/efi.img |  | El Torito boot image for EFI |
|-no-emul-boot  |  | Again??|
|-isohybrid-gpt-basdat | | boot image is GPT partition with EFI Master Boot Record|
|-o |"$TARGET_ISO" ./|  sets the ISO file name|



