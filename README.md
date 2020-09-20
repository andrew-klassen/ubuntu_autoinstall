# ubuntu_autoinstall
Below are instructions for how to create an automated ubuntu iso.

1. Download the latest ubuntu live server iso and move the iso to the ubuntu machine you want to use to build the iso. You can use wget to download the iso from the command line like so.

     `wget https://mirror.us.leaseweb.net/ubuntu-releases/20.04.1/ubuntu-20.04.1-live-server-amd64.iso`

2. Download and install some needed dependencies.

     **xorriso** - tool for building the iso image
     
     **isolinux** - needed for making the iso uefi bootable

     ```
     apt update
     apt install xorriso isolinux
     ```
   
3. Create two new directories. One of them is to mount the ubuntu iso you just downloaded, the other is to work out of.

     `mkdir temp_iso_mount iso_files`
     
4. Mount the iso image. It will mount as read-only.

     `mount -t iso9660 ubuntu-20.04.1-live-server-amd64.iso temp_iso_mount`
     
5. Copy the files from the iso to the working directory.

     `cp -rT temp_iso_mount iso_files`
    
6. Edit the **iso_files/isolinux/txt.cfg** and add a new menu entry. The entire file should look like the following. I added the **auto** entry. Make sure to change the default to the new entry/label you just created. This file takes effect for legacy booting. If the file is read only use **chmod 666** to make it writable.

     ```
     default auto
     label auto
       menu label ^Install Ubuntu Server (auto)
       kernel /casper/vmlinuz
       append   initrd=/casper/initrd quiet  --- autoinstall ds=nocloud;s=/cdrom/autoinstall/
     label live
       menu label ^Install Ubuntu Server (manual)
       kernel /casper/vmlinuz
       append   initrd=/casper/initrd quiet  ---
     label live-nomodeset
       menu label ^Install Ubuntu Server (safe graphics)
       kernel /casper/vmlinuz
       append   initrd=/casper/initrd quiet  nomodeset ---
     label memtest
       menu label Test ^memory
       kernel /install/mt86plus
     label hd
       menu label ^Boot from first hard disk
       localboot 0x80
     ```
     
7. Edit the **iso_files/boot/grub/grub.cfg** and add a new menu entry. The entire file should like the following. The menu entry I added is **Install Ubuntu Server (auto)**. This file is used for UEFI booting. If the file is read only use **chmod 666** to make it writable.

     ```
     if loadfont /boot/grub/font.pf2 ; then
             set gfxmode=auto
             insmod efi_gop
             insmod efi_uga
             insmod gfxterm
             terminal_output gfxterm
     fi

     set menu_color_normal=white/black
     set menu_color_highlight=black/light-gray

     set timeout=5
     menuentry "Install Ubuntu Server (auto)" {
             set gfxpayload=keep
             linux   /casper/vmlinuz "ds=nocloud;s=/cdrom/autoinstall/"  quiet autoinstall ---
             initrd  /casper/initrd
     }
     menuentry "Install Ubuntu Server (manual)" {
             set gfxpayload=keep
             linux   /casper/vmlinuz quiet ---
             initrd  /casper/initrd
     }
     menuentry "Install Ubuntu Server (safe graphics)" {
             set gfxpayload=keep
             linux   /casper/vmlinuz   quiet  nomodeset ---
             initrd  /casper/initrd
     }
     grub_platform
     if [ "$grub_platform" = "efi" ]; then
     menuentry 'Boot from next volume' {
             exit
     }
     menuentry 'UEFI Firmware Settings' {
             fwsetup
     }
     fi
     ```
     
8. Make a directory in the working directory for the autoinstall files and then make an empty meta-data file inside of it.

     ```
     mkdir iso_files/autoinstall
     touch iso_files/autoinstall/meta-data
     ```
     
9. Create and edit **iso_files/autoinstall/user-data**, this is the file that provides the answers to the installer.

     `nano iso_files/autoinstall/user-data`
     
Below is a complete example of a working config file. The comment and version 1 at the top are nessasary and will break the installation if not present. You can use **openssl passwd -6 "the_password"** to generate a sha-512 hash for the password value.

```
#cloud-config
autoinstall:
  version: 1
  early-commands:
    - systemctl stop ssh
  ssh:
    install-server: yes
    allow-pw: yes
  locale: en_US.UTF-8
  keyboard:
    layout: en
  identity:
    hostname: changeme
    password: "$6$mYWDL1/gejskwkK0$Aloc4nYx5lJUw2E5E.cm9LpWmdL5RBAH5AqxlIp1DGvtRrXdefsIUvC3psWSryI8x9Ez/NMC.ej.Oh9Rk.3NU0" # root
    username: admin
  packages:
    - vim
    - git
  user-data:
    disable_root: false
  late-commands:
    - echo 'sudo ALL=(ALL) NOPASSWD:ALL' > /target/etc/sudoers.d/ubuntu
    - timedatectl set-timezone America/Los_Angeles
    - sed -i "/^\/swap.img/d" /target/etc/fstab
    - rm -f /target/swap.img
    - shutdown -P now 
```
      
10. Use the following command to build the iso.

     ```
     xorriso -as mkisofs -isohybrid-mbr /usr/lib/ISOLINUX/isohdpfx.bin -c isolinux/boot.cat -b isolinux/isolinux.bin -no-emul-boot -boot-load-size 4 -boot-info-table -eltorito-alt-boot -e boot/grub/efi.img -no-emul-boot -isohybrid-gpt-basdat -o ubuntu-20.04-autoinstall.iso iso_files
     ```
     
