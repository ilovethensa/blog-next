---
title: "How to install encrypted root-on-tmpfs nixos"
draft: false
date: 2024-02-27
description: "Guide on how to install nixos on tmpfs encrypted"
---
- This guide assumes that you have already booted into NixOS live.
- Basic Linux knowledge is assumed.

This guide is mainly based off of [this blog post](https://elis.nu/blog/2020/05/nixos-tmpfs-as-root/) but tweaked for Btrfs and encryption

## Partitioning the drives
```bash
cfdisk /dev/sda
```

## Creating filesystems
Firstly you need to create a 32-bit vFAT for the ESP partition
```bash
mkfs.vfat -F32 /dev/sda1
```
Then we can create a LUKS parition for /nix
```bash
# Format the drive to LUKS
cryptsetup luksFormat /dev/sda2
# Open the LUKS drive
cryptsetup luksOpen /dev/sda2 cryptroot
mkfs.btrfs /dev/mapper/cryptroot
```

## Creating subvolumes
```bash
# Temporally mount the partition to /mnt
mount /dev/mapper/cryptroot /mnt
# Create a subvolume for /nix
btrfs subvolume create /mnt/@nix
# Create a subvolume for /home
btrfs subvolume create /mnt/@home
# Unmount it
umount /mnt
```

# Setting up the filesystems
```bash
# Mount the root file system
mount -t tmpfs none /mnt
# Create directories
mkdir -p /mnt/{boot,nix,home}

# Mount /boot and /nix
mount -t btrfs -o compress=zstd,subvol=@nix /dev/mapper/cryptroot /mnt/nix
mount -t btrfs -o compress=zstd,subvol=@home /dev/mapper/cryptroot /mnt/home
mount /dev/sda1 /mnt/boot
```

## Configuration
Now first create a nixos config using
```bash
nixos-generate-config --root /mnt
```

```nix
fileSystems."/" = {
  fsType = "tmpfs";
  options = [ "defaults" "size=2G" "mode=755" ];
};
```
### Users
When you have a system with a tmpfs root you have to configure all users and passwords in configuration.nix, otherwise you won’t have any user or a password on the next boot.

You probably want to have immutable users as well since it doesn’t make any sense to have mutability of users if it’s going to reset anyways.

>Note: Don’t use the options password or hashedPassword for users because it won’t work. It has to be the options named initialPassword or initialHashedPassword.
```nix
{
  # …

  # Don't allow mutation of users outside of the config.
  users.mutableUsers = false;

  # Set a root password, consider using initialHashedPassword instead.
  #
  # To generate a hash to put in initialHashedPassword
  # you can do this:
  # $ nix-shell --run 'mkpasswd -m SHA-512 -s' -p mkpasswd
  users.users.root.initialPassword = "hunter2";

  # …
}
```
### Other
You can now tweak anything else you want in the config file(Timezones, users, etc)
Just make sure to copy `/etc/nixos` to `/nix` to preserve your configs

## Installing
Now to install the system just run
```
nixos-install --no-root-passwd
```