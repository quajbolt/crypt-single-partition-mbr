<!-- ABOUT THE PROJECT -->
## About The Project

![Explaining]

Initial ramdisk is used for lots of things by linux server admins by variety things, this is also very valid usecase.
LVM (Logical Volume Manager) and cryptsetup are two powerful tools in the Linux ecosystem that complement each other seamlessly, providing both flexibility in storage management and robust security through encryption.

There's some key features that you gain with lvm and cryptsetup!
  - **Dynamic Resizing:** With LVM, you can easily resize your logical volumes as your storage needs change.
  - **Layered Architecture:** LVM operates at a higher level, managing logical volumes, while cryptsetup works at the block device level. This layered approach allows users to create encrypted logical volumes that can be resized or modified without compromising security.
  - **Snapshots and Backups:** LVM supports snapshots, allowing users to create point-in-time copies of their logical volumes.
  - **Ease of Management:** Both tools are designed to be user-friendly, with straightforward command-line interfaces.



<!-- GETTING STARTED -->
## Getting Started

This is an example of how you may give instructions on setting up your project locally.
To get a local copy up and running follow these simple example steps.



### Prerequisites

This is an example of how to list things you need to use the software and how to install them.
* debian or ubuntu
  ```sh
  sudo apt-get install util-linux coreutils lvm2 cryptsetup mkinitcpio grub
  ```

* arch based
  ```sh
  sudo pacman -Syy util-linux coreutils lvm2 cryptsetup mkinitcpio grub
  ```
*Note*: util-linux package contains fdisk binary and coreutils package contains sha512sum which explains why it's used here



### Installation

*Note 2*: I'm gonna use /dev/sdX for demonstration. Please change it for your disk.

1. let's remove all the remaining partition from our disk
   ```sh
   sudo fdisk /dev/sdX
   ```
 - Type "d" to delete a partition. You will be prompted to specify the partition number. Repeat this step for each partition on the disk.
 - Alternatively, you can type "g" to create a new empty GPT partition table, which will remove all existing partitions.

2. Create a MBR partition at fdisk interactive prompt. Type "o" for it.
3. Type "w" to write changes to disk. not exiting yet.
4. Now let's create out single partition. Type "n" to create new partition.
5. Type "p" for primary partition.
6. Next, you will be prompted for the partition number. You can usually just press Enter to accept the default (which is typically 1).
7. For the first sector, you can press Enter to accept the default (which is usually the start of the disk).
8. For the last sector, you can press Enter again to use all available space on the disk.
9. Type "w" to write changes to disk. Then type "q" to exit.


Now we created our partition successfully with able to unlock using grub. Next we'll create LVM partitions and and get 2 child partitions by doing that (boot and rootfs).

1. Let's encrypt our partition first
```sh
sudo cryptsetup luksFormat --cipher aes-xts-plain64 --key-size 512 --hash sha512 /dev/sdX0
```
2. Opening it with:
```sh
sudo cryptsetup luksOpen /dev/sdX0 matrix
```
4. Now we need to create out LVM partitions
Create a Physical Volume (PV):
```sh
sudo pvcreate /dev/mapper/matrix
```
Create a Volume Group (VG):
```sh
sudo vgcreate matrix /dev/mapper/matrix
```
Create Logical Volumes (LV):
- For the boot partition (2 GB):
  ```sh
  sudo lvcreate -L 2G -n lv_boot matrix
  ```
- For the root filesystem (remaining space):
  ```sh
  sudo lvcreate -l +100%FREE -n lv_root matrix
  ```
4. Format the Logical Volumes:
```sh
sudo mkfs.ext4 /dev/matrix/lv_boot
sudo mkfs.ext4 /dev/matrix/lv_root
```

Now you need to mount them and keep installing your distro. but for now I'm gonna show an example for mounting. Setup part is up to user.
```sh
sudo mount /dev/matrix/lv_root /mnt
sudo mkdir -p /mnt/boot
sudo mount /dev/matrix/lv_boot /mnt/boot
```


## Automatic encryption after password passed from grub

Now after all that, we need a usable system. For that purpose we're gonna use mkinitcpio to contain our keyfile at initial ramdisk. that'll help us to unlock our LVM partitions after grub menu. it is crucial because even if we pass our password to grub, our kernel doesn't know how to jump into our real rootfs since it's encrypted.

Let's start with fstab
  1. Let's get our UUID's for lv_boot and lv_root
  ```sh
  blkid | grep "/dev/mapper/matrix-lv_boot"
  blkid | grep "/dev/mapper/matrix-lv_root"
  ```
  2. Now we need to pass the UUID's to /etc/fstab file. End result it should look like that:
  ```sh
  # /dev/mapper/matrix-lv_root
  UUID=46792b5e-3a62-4bc5-b2f3-2bf5b97c1531       /               ext4            rw,relatime     0 1

  # /dev/mapper/matrix-lv_boot
  UUID=8d22c4cc-dfc8-48b1-8006-aab2fd9fa6c2       /boot           ext4            rw,relatime     0 2
  ```

  3. Let's proceed to creating next file which is /etc/crypttab :
  ```sh
  matrix UUID=4628e2e9-ad41-4d54-b193-3136852a4898        /keyfile        luks
  ```
  - As you can see, there's a "/keyfile" and UUID that belongs to /dev/sdX0

  4. For the last, we need a keyfile. There's how you can create and associate with your encrypted partition:
  ```sh
  sudo dd if=/dev/urandom of=/keyfile bs=64 count=1
  sudo sha512sum /keyfile
  sudo chmod 600 /keyfile
  sudo cryptsetup luksAddKey /dev/sdX0 /keyfile
  ```

  Now our kernel are able to encrypt when using systemd, but of course you shouldn't trust system management tools because when they become faulty, you'll be not able to access your system. For that reason we'll gonna create a script that unlocks our partition and this script will gonna be a mkinitcpio hook with file=(/keyfile). Here's how to do it:

  1. First we're gonna create our hook named "decryptmod"
  ```sh
  sudo touch /usr/lib/initcpio/install/decryptmod
  sudo echo "#!/bin/bash" > /usr/lib/initcpio/install/decryptmod
  sudo echo "build(){" >> /usr/lib/initcpio/install/decryptmod
  sudo echo "add_runscript" >> /usr/lib/initcpio/install/decryptmod
  sudo echo "}" >> /usr/lib/initcpio/install/decryptmod
  sudo cat /usr/lib/initcpio/install/decryptmod
  ```
  output should be :
  ```sh
  #!/bin/bash
  build(){
  add_runscript
  }
  ```

  2. Let's create our hook's functionallity with:
  ```sh
  sudo touch /usr/lib/initcpio/hooks/decryptmod
  sudo echo "#!/bin/bash" > /usr/lib/initcpio/hooks/decryptmod
  sudo echo "run_hook(){" >> /usr/lib/initcpio/hooks/decryptmod
  sudo echo "cryptsetup open /dev/sdX0 matrix --key-file /keyfile" >> /usr/lib/initcpio/hooks/decryptmod
  sudo echo "}" >> /usr/lib/initcpio/hooks/decryptmod
  sudo cat /usr/lib/initcpio/hooks/decryptmod
  ```
  output should be :
  ```sh
  #!/bin/bash
  run_hook(){
  cryptsetup open /dev/sdX0 matrix --key-file /keyfile
  }
  ```

  3. Lastly we need to edit /etc/mkinitcpio.conf to show where to look for /keyfile and make use of our brand new "decryptmod" hook
  ```sh
  sudo nano /etc/mkinitcpio.conf
  ```
  - Edit this: ```FILES=()``` to this: ```FILES=(/keyfile)```
  - Find uncommented ```HOOKS=``` and add this (after "block or consolefont"): ```encrypt lvm2 decryptmod```

  You just created a new hook and also maked your initramfs able to encrypt partition and know which partition is LVM. Also there's auto decryption on top all of that.



## Finalizing with grub config and generating everything to boot our new system
  1. Let's edit our grub by typing:
  ```sh
  sudo nano /etc/default/grub
  ```
  to the terminal.

  2. Find ```GRUB_ENABLE_CRYPTODISK``` and change it's value to: ```y```
  3. Find ```GRUB_PRELOAD_MODULES``` and add these to it's value: ```cryptodisk luks lvm```
  4. For the last. We need to find out our /dev/sdX0 UUID. which we found beforehand at the guide.
  5. With knowing your UUID. ```Find GRUB_CMDLINE_LINUX``` and add these to it's value. make sure to use your own UUID always
  ```sh
  "cryptodevice=UUID=4628e2e9-ad41-4d54-b193-3136852a4898:matrix root=/dev/mapper/matrix-lv_root"
  ```

  Now we can generate what we need and reboot into our new system
  ```sh
  sudo mkinitcpio -P
  sudo grub-install /dev/sdX
  sudo grub-mkconfig -o /boot/grub/grub.cfg
  ```

  This is the end of guide.


<!-- CONTRIBUTING -->
## Contributing

Contributions are what make the open source community such an amazing place to learn, inspire, and create. Any contributions you make are **greatly appreciated**. Even typos!

If you have a suggestion that would make this better, please fork the repo and create a pull request. You can also simply open an issue with the tag "enhancement".
Don't forget to give the guide a star! Thanks again!

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request


<!-- LICENSE -->
## License

Distributed under the CC0-1.0 license. See `LICENSE` for more information.


<!-- CONTACT -->
## Contact

Project Link: [https://github.com/quajbolt/crypt-single-partition-mbr](https://github.com/quajbolt/crypt-single-partition-mbr)



<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->
[Explaining]: https://raw.githubusercontent.com/quajbolt/crypt-single-partition-mbr/refs/heads/main/drawing0.svg
