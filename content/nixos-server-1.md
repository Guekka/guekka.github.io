+++
title = "NixOS as a server, part 1: Impermanence"
date = 2023-02-20
draft = false

[taxonomies]
categories = ["Projects"]
tags = ["nix", "self-hosting" ]
+++

A few months ago, I woke up with the idea of hosting my own services. I went through a lot of tries. LXC, Debian, Alpine, (rootless or not) Docker, podman, portainer...

But no solution felt perfect. Until I decided to have a try at hosting using NixOS.

I'm going to assume you know about NixOS and have some prior experience. However, for a small summary: NixOS is a Linux distribution revolving around the Nix package manager. Its main advantage is having a reproducible environment through a declarative configuration. This means that you can copy an entire computer configuration easily: if it works somewhere, it will work anywhere.

I had been using NixOS as a desktop distribution, so it was time to use it as a server too. My main focus point is reproducibility, so that's why we'll start with configuring *impermanence*.

## What's impermanence?

Originally, a philosophic concept. But in our case, impermanence means erasing the `/` drive at each reboot. You heard that right, erasing *almost* everything at each reboot. I'm not going to explain this part in much detail, as others have done it before me:
- [Erase your darlings: immutable infrastructure for mutable systems - Graham Christensen](https://grahamc.com/blog/erase-your-darlings)
- [NixOS ❄: tmpfs as root](https://elis.nu/blog/2020/05/nixos-tmpfs-as-root/)
- [Encypted Btrfs Root with Opt-in State on NixOS](https://mt-caret.github.io/blog/posts/2020-06-29-optin-state.html#fn6)
- [Paranoid NixOS Setup - Xe Iaso](https://xeiaso.net/blog/paranoid-nixos-2021-07-18)
- [nix-community/impermanence: NixOS module](https://github.com/nix-community/impermanence)

The goal is the following: over years, configuration files accumulate. Sometimes editing `/etc` is required, because of a bug or an obscure configuration. NixOS allows us to avoid this manual file editing, but it does not *force* us to do so. We can still have a lot of important state, breaking the reproducibility promise.

So what can we do instead? Erase everything, at each reboot. This way, we'll be sure the only source of truth is our configuration.

## Installing the system

I'm currently using a [quickemu](https://github.com/quickemu-project/quickemu) VM. This is not a recommenced setup and is only done for testing. Configuration file:
```conf
#!/usr/bin/quickemu --vm
guest_os="linux"
disk_img="nixos-22.11-minimal/disk.qcow2"
iso="nixos-22.11-minimal/latest-nixos-minimal-x86_64-linux.iso"
disk_size="50G"
ram="4G"
```
Let's first format it:
```sh
DISK=/dev/vda

parted "$DISK" -- mklabel gpt
parted "$DISK" -- mkpart ESP fat32 1MiB 1GiB
parted "$DISK" -- set 1 boot on
mkfs.vfat "$DISK"1
```
```sh
parted "$DISK" -- mkpart Swap linux-swap 1GiB 9GiB
mkswap -L Swap "$DISK"2
swapon "$DISK"2
```
> Using swap in 2023!?

[Yes](https://chrisdown.name/2018/01/02/in-defence-of-swap.html).
```sh
parted "$DISK" -- mkpart primary 9GiB 100%
mkfs.btrfs -L Butter "$DISK"3
```
While the `impermanence` module recommends using `tmpfs` for `/`, I chose to use `btrfs`: I do not have RAM to waste. Furthermore, this will allow us to use a nice script we'll see later on.

Let's create `btrfs` subvolumes:
```sh
mount "$DISK"3 /mnt
btrfs subvolume create /mnt/root
btrfs subvolume create /mnt/home
btrfs subvolume create /mnt/nix
btrfs subvolume create /mnt/persist
btrfs subvolume create /mnt/log
```
And now, the crucial part:
```sh
btrfs subvolume snapshot -r /mnt/root /mnt/root-blank
```
We just took a snapshot of that empty volume. We will restore it at each reboot.
We can now mount the subvolumes and let `nix-generate-config` do its job
```sh
mount -o subvol=root,compress=zstd,noatime "$DISK"3 /mnt

mkdir /mnt/home
mount -o subvol=home,compress=zstd,noatime "$DISK"3 /mnt/home

mkdir /mnt/nix
mount -o subvol=nix,compress=zstd,noatime "$DISK"3 /mnt/nix

mkdir /mnt/persist
mount -o subvol=persist,compress=zstd,noatime "$DISK"3 /mnt/persist

mkdir -p /mnt/var/log
mount -o subvol=log,compress=zstd,noatime "$DISK"3 /mnt/var/log

mkdir /mnt/boot
mount "$DISK"1 /mnt/boot

nixos-generate-config --root /mnt
```
Lastly, we only have to edit the generated configuration files at `/mnt/etc/nixos`.

My final configuration is available [here](https://github.com/Guekka/nixos-server/tree/1-impermanence). You can follow all the steps by looking at the [commits](https://github.com/Guekka/nixos-server/commits/1-impermanence-tailscale).

## Configuring the system

- Checking that we have the correct mount options in `/mnt/etc/nixos/hardware-configuration.nix`.
I added `"compress=zstd" "noatime"` to all filesystems. We also need to add `neededForBoot` to `/var/log` and `/persist`.

- replace default values in `configuration.nix`
I've enabled `networkmanager`, removed most suggested options and enabled `system.copySystemConfiguration`.

This last option copies the current configuration to `/run/current_system/configuration.nix`. You should not rely on it: keep your configuration in a git repository. But it can serve as some kind of last chance.

- declaring a user, including ssh
```nix
users.mutableUsers = false;
users.users.user = {
 isNormalUser = true;
 extraGroups = [ "wheel" ];

 openssh.authorizedKeys.keys = [ "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICWVNch9BcjkMqS/Xwep+GN4HwqyRIjr3Cuw7mHpqsKr nixos" ];

 # passwordFile needs to be in a volume marked with `neededForBoot = true`
 passwordFile = "/persist/passwords/user";
};
```
Here we have completely disabled imperative user modification. This does not matter much, as imperative changes would be erased anyway at start.
We thus need to provide a password. We're using `passwordFile` for that: a path to a file containing the hashed password.

Here's how to generate that file: `sudo mkpasswd -m sha-512 "hunter2" > /mnt/persist/passwords/user`.

The SSH key was generated using `ssh-keygen -t ed25519 -C "nixos".

- Enabling openSSH
We're going to use Xe's configuration:
```nix
  services.openssh = {
   enable = true;
   passwordAuthentication = false;
   allowSFTP = false; # Don't set this if you need sftp
   challengeResponseAuthentication = false;
   extraConfig = ''
     AllowTcpForwarding yes
     X11Forwarding no
     AllowAgentForwarding no
     AllowStreamLocalForwarding no
     AuthenticationMethods publickey
   '';
  };
```
This reduces attack surface, for example by disabling stream-local forwarding and disabling password authentification.
***
This will be enough for now. Let's install the system before going to the next step: `sudo nixos-install --root /mnt && sudo reboot`. You should be able to connect by SSH using the previously defined key, or login using the password you defined in `/persist/passwords/user`.

## Configuring impermanence

We've created our volumes, we've configured the system... But I promised we would reset our system at each reboot. Let's do that now!
We're going to use the following script, credit of mt-caret. Do not forget to replace `/dev/vda3` with your data partition.
```nix
  # Note `lib.mkBefore` is used instead of `lib.mkAfter` here.
  boot.initrd.postDeviceCommands = pkgs.lib.mkBefore ''
    mkdir -p /mnt

    # We first mount the btrfs root to /mnt
    # so we can manipulate btrfs subvolumes.
    mount -o subvol=/ /dev/vda3 /mnt

    # While we're tempted to just delete /root and create
    # a new snapshot from /root-blank, /root is already
    # populated at this point with a number of subvolumes,
    # which makes `btrfs subvolume delete` fail.
    # So, we remove them first.
    #
    # /root contains subvolumes:
    # - /root/var/lib/portables
    # - /root/var/lib/machines
    #
    # I suspect these are related to systemd-nspawn, but
    # since I don't use it I'm not 100% sure.
    # Anyhow, deleting these subvolumes hasn't resulted
    # in any issues so far, except for fairly
    # benign-looking errors from systemd-tmpfiles.
    btrfs subvolume list -o /mnt/root |
    cut -f9 -d' ' |
    while read subvolume; do
      echo "deleting /$subvolume subvolume..."
      btrfs subvolume delete "/mnt/$subvolume"
    done &&
    echo "deleting /root subvolume..." &&
    btrfs subvolume delete /mnt/root

    echo "restoring blank /root subvolume..."
    btrfs subvolume snapshot /mnt/root-blank /mnt/root

    # Once we're done rolling back to a blank snapshot,
    # we can unmount /mnt and continue on the boot process.
    umount /mnt
  '';
```
We can then specify the files we want to keep.

So, which files do we want to keep? Let's find out. Thanks to another useful script of mt-caret, we can list the differences between our current `/` and the blank state:
```sh
#!/usr/bin/env bash
# fs-diff.sh
set -euo pipefail

OLD_TRANSID=$(sudo btrfs subvolume find-new /mnt/root-blank 9999999)
OLD_TRANSID=${OLD_TRANSID#transid marker was }

sudo btrfs subvolume find-new "/mnt/root" "$OLD_TRANSID" |
sed '$d' |
cut -f17- -d' ' |
sort |
uniq |
while read path; do
  path="/$path"
  if [ -L "$path" ]; then
    : # The path is a symbolic link, so is probably handled by NixOS already
  elif [ -d "$path" ]; then
    : # The path is a directory, ignore
  else
    echo "$path"
  fi
done
```
Used like this:
```sh
sudo mkdir /mnt ; sudo mount -o subvol=/ /dev/vda3 /mnt ; ./fs-diff.sh
```
Here's the result of mine:
```raw
/etc/.clean
/etc/group
/etc/machine-id
/etc/nixos/configuration.nix
/etc/nixos/hardware-configuration.nix
/etc/passwd
/etc/resolv.conf
/etc/shadow
/etc/ssh/authorized_keys.d/user
/etc/ssh/ssh_host_ed25519_key
/etc/ssh/ssh_host_ed25519_key.pub
/etc/ssh/ssh_host_rsa_key
/etc/ssh/ssh_host_rsa_key.pub
/etc/subgid
/etc/subuid
/etc/sudoers
/etc/.updated
/root/.nix-channels
/root/.nix-defexpr/channels
/var/lib/NetworkManager/internal-84e273c2-b91a-3a96-b341-8234a339bdc7-enp0s8.lease
/var/lib/NetworkManager/internal-84e273c2-b91a-3a96-b341-8234a339bdc7-enp0s9.lease
/var/lib/NetworkManager/NetworkManager-intern.conf
/var/lib/NetworkManager/secret_key
/var/lib/NetworkManager/timestamps
/var/lib/nixos/auto-subuid-map
/var/lib/nixos/declarative-groups
/var/lib/nixos/declarative-users
/var/lib/nixos/gid-map
/var/lib/nixos/uid-map
/var/lib/systemd/catalog/database
/var/lib/systemd/random-seed
/var/.updated
```
That's not too bad!

Out of these, there's almost nothing I want to preserve.

Let's make use of the `impermanence` module. We need to download the module:
```nix
let
  impermanence = builtins.fetchTarball "https://github.com/nix-community/impermanence/archive/master.tar.gz";
in
{
imports = [ "${impermanence}/nixos.nix" ./hardware-configuration.nix ]
// the whole configuration
}
```
And now, we can just tell it the files and directories that we want:
```nix
  # configure impermanence
  environment.persistence."/persist" = {
    directories = [
      "/etc/nixos"
    ];
    files = [
      "/etc/machine-id"
      "/etc/ssh/ssh_host_ed25519_key"
      "/etc/ssh/ssh_host_ed25519_key.pub"
      "/etc/ssh/ssh_host_rsa_key"
      "/etc/ssh/ssh_host_rsa_key.pub"
  };

  security.sudo.extraConfig = ''
    # rollback results in sudo lectures after each reboot
    Defaults lecture = never
  '';
```
What an ergonomic interface.

> Wait, did you just say Nix was ergonomic?

Well, yes. Sometimes.
***
I have not saved my network manager configuration, but you may need to.

When new files are set to be preserved, **it is necessary to copy them manually to `/persist`**:
```sh
sudo nixos-rebuild boot

sudo mkdir /persist/etc

sudo cp -r {,/persist}/etc/nixos
sudo cp {,/persist}/etc/machine-id

sudo mkdir /persist/etc/ssh

sudo cp {,/persist}/etc/ssh/ssh_host_ed25519_key
sudo cp {,/persist}/etc/ssh/ssh_host_ed25519_key.pub
sudo cp {,/persist}/etc/ssh/ssh_host_rsa_key
sudo cp {,/persist}/etc/ssh/ssh_host_rsa_key.pub
```
Now, if we reboot and list files again:
```raw
/etc/.clean
/etc/group
/etc/passwd
/etc/resolv.conf
/etc/shadow
/etc/ssh/authorized_keys.d/user
/etc/subgid
/etc/subuid
/etc/sudoers
/etc/.updated
/root/.nix-channels
/var/lib/NetworkManager/internal-84e273c2-b91a-3a96-b341-8234a339bdc7-enp0s9.lease
/var/lib/NetworkManager/NetworkManager-intern.conf
/var/lib/NetworkManager/secret_key
/var/lib/NetworkManager/timestamps
/var/lib/nixos/auto-subuid-map
/var/lib/nixos/declarative-groups
/var/lib/nixos/declarative-users
/var/lib/nixos/gid-map
/var/lib/nixos/uid-map
/var/lib/systemd/catalog/database
/var/lib/systemd/random-seed
/var/.updated
```
Success! The files we persisted are no longer showing up.

### What about our home directory?

It is possible to setup the impermanence module for our home directory. However, I did not want to go through `home-manager` installation. Furthermore, a home directory is meant to be stateful.

In our case, we are creating a server, so it would still make sense to configure it. If you are interested, I will direct you to [tmpfs at home](https://elis.nu/blog/2020/06/nixos-tmpfs-as-home/)

## Conclusion

Todo

