---
title: "Building a NixOS-Based NAS"
date: 2022-07-24
draft: false
tags: [nixos, zfs]
---

I recently got around to fixing an old Ubuntu-based NAS that I set up years ago. It was initially set up before I was using configuration management tools like Ansible and had a number of issues that made it nearly refuse to boot. This called for a full reinstall and I thought I'd try to build it all with NixOS given my recent interest in it. I thought I'd use this opportunity to actually set up everything properly (with the caveat of being compatible with my old arrays).

## Installing NixOS

### ZFS All the Way Down

Since my old system was ZFS-based, I decided to go with ZFS for all my filesystems. I roughly followed the instructions in the [NixOS wiki][nixos-zfs]:

```bash
# Set this to your root partition
export ROOT_PART=/dev/disk/by-id/...
sudo cryptsetup luksFormat $ROOT_PART
sudo cryptsetup open --type luks $ROOT_PART crypt-root
sudo zpool create -O mountpoint=none nixos-rpool /dev/mapper/crypt-root
# Make a reservation since ZFS is copy on write and will explode
# if we try to delete files after running out of space
sudo zfs create -o refreservation=1G -o mountpoint=none nixos-rpool/reserve
sudo zfs create \
  -o xattr=sa \
  -o acltype=posixacl \
  -o compression=zstd \
  -o relatime=on \
  -o mountpoint=/ \
  -o canmount=on \
  nixos-rpool/nixos
sudo zfs create nixos-rpool/nixos/nix
sudo zfs create -o mountpoint=legacy nixos-rpool/nixos/root
```

Note that if you make a mistake in setting `mountpoint=legacy`, the ZFS dataset will automatically get mounted to `/`, shadow the installer's nix store, and break the install, causing you to start over from the beginning!

#### Compression and Deduplication

I'm no expert in ZFS and was not sure if I should enable compression or deduplication for the root datasets. After a number of weeks with this setup (and using it as my main host for [morph][morph] deployments), the compression on the root volume seems to be quite useful:

```shell
$ zfs get compressratio
NAME                    PROPERTY       VALUE  SOURCE
nixos-rpool             compressratio  1.19x  -
nixos-rpool/nixos       compressratio  1.19x  -
nixos-rpool/nixos/nix   compressratio  2.12x  -
nixos-rpool/nixos/root  compressratio  1.14x  -
nixos-rpool/reserve     compressratio  1.00x  -
```

Additionally, I have been using deduplication on the root volume. This has also stretched my root nVME drive a bit further:

```shell
$ zpool list -v nixos-rpool
NAME           SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
nixos-rpool    472G   160G   312G        -         -    17%    33%  1.24x    ONLINE  -
$ zpool status -D nixos-rpool
  pool: nixos-rpool
 state: ONLINE
config:

        NAME          STATE     READ WRITE CKSUM
        nixos-rpool   ONLINE       0     0     0
          crypt-root  ONLINE       0     0     0

errors: No known data errors

 dedup: DDT entries 1877158, size 324B on disk, 181B in core

bucket              allocated                       referenced          
______   ______________________________   ______________________________
refcnt   blocks   LSIZE   PSIZE   DSIZE   blocks   LSIZE   PSIZE   DSIZE
------   ------   -----   -----   -----   ------   -----   -----   -----
     1    1.46M    154G    128G    128G    1.46M    154G    128G    128G
     2     326K   30.0G   26.9G   26.9G     749K   70.7G   64.0G   64.0G
     4    12.4K    494M    185M    185M    56.4K   2.32G    871M    871M
     8    1.51K   53.6M   29.9M   29.9M    17.2K    572M    290M    290M
    16      549   3.55M   2.37M   2.37M    9.19K   64.7M   44.6M   44.6M
    32       43   1.07M    806K    806K    1.79K   37.1M   27.2M   27.2M
    64       15   10.5K   8.50K   8.50K    1.28K    944K    760K    760K
   128        2      1K      1K      1K      334    167K    167K    167K
   256        4   5.50K   3.50K   3.50K    1.78K   2.44M   1.55M   1.55M
    1K        1    512B    512B    512B    1.21K    621K    621K    621K
 Total    1.79M    184G    155G    155G    2.28M    228G    193G    193G
```

### Encryption

As seen in [ZFS All the Way Down](#zfs-all-the-way-down), I initially created LUKS volumes before creating the ZFS datasets. This is to prevent reading of the drive if anyone were to physically obtain my server's drive.

I went for LUKS encryption over ZFS-native encryption because my old NAS was built using LUKS over ZFS (ZFS-native encryption didn't exist back then).

#### Root

NixOS allows pretty easy setup for regular LUKS devices via `boot.initrd.luks.devices`, providing you want to walk over to your keyboard to unlock on boot:

```nix
boot.loader.grub.enableCryptodisk = true;
boot.loader.grub.zfsSupport = true;
boot.initrd.luks.forceLuksSupportInInitrd = true;
boot.initrd.luks.devices = {
  crypt-root = {
    device = "/dev/disk/by-id/...";
    preLVM = true;
  };
};
```

However, I didn't want to go over to my server to unlock on every boot and wanted to be able to reboot while not physically at my apartment. 

While I think a solution like [pikvm][pikvm] would be a better solution here (lack of cryptographic holes and ability to boot into previous NixOS generations if something goes wrong), I followed a couple guides[^remote-decryption][^nixos-wiki-remote-decryption] to set up an SSH server during the init phase where I can unlock the root volume.

After a lot of trial and error (and being saved by NixOS generations), I came up with this:

```nix
# Enable SSH in initrd so we can SSH in and unlock the volume
# Adapted from https://mth.st/blog/nixos-initrd-ssh/
boot.initrd.network = {
  enable = true;
  ssh = {
    enable = true;
    # Use a different port so we won't always have host key conflicts
    port = 2222;
    authorizedKeys = config.users.users.daniel.openssh.authorizedKeys.keys;      
    # Note that these will probably be unencrypted in our setup, but it's mostly fine
    hostKeys = [
      "/etc/secrets/initrd/ssh_host_rsa_key"
      "/etc/secrets/initrd/ssh_host_ed25519_key"
    ];
  };
      
    # Set the shell profile to meet SSH connections with a decryption
    # prompt that writes to /tmp/continue if successful.
    postCommands =
    let
      disk = "/dev/disk/by-id/...";
    in
    ''
echo 'cryptsetup open ${disk} crypt-root --type luks && echo > /tmp/continue && exit' >> /root/.profile
echo 'starting sshd...'
    '';
};


# Even though the device is marked as neededForBoot, it doesn't seem to mount in time to decrypt
# the storage/media disks
# Adapted from https://nixos.wiki/wiki/Full_Disk_Encryption#Option_2:_Copy_Key_as_file_onto_a_vfat_usb_stick
boot.initrd.postDeviceCommands = pkgs.lib.mkBefore ''
echo 'waiting for root device to be opened...'
mkfifo /tmp/continue
cat /tmp/continue
mkdir -m 0755 ${keyMount} # This will go away after stage 1
mkdir -m 0755 /key-mount
sleep 2
echo 'mounting key volume'
mount -n -t ext4 -o ro `findfs UUID=953ef32c-e9cb-4049-bf33-56f9e2c84b55` /key-mount
cp /key-mount/etc/apollo.kf /key
cp /key-mount/etc/keyfile /key
umount /key-mount # This will make fsck work
rmdir /key-mount
'';
```

Basically, we enable an SSH server in the init process (thankfully, NixOS makes this easy with `boot.initrd.network.ssh`) and we create a FIFO at `/tmp/continue` that will pause boot until it is written to. By default, when an SSH session starts, the script
`cryptsetup open ${disk} crypt-root --type luks && echo > /tmp/continue && exit` runs and `cryptsetup` waits for passphrase input. Note that you can Ctrl-C out of this to get a shell, though it is extremely limited at this point.

Note that there is one cryptographic hole in this setup: initrd needs to be readable for the boot process and contains the host keys for the initrd SSH server (the keys under `/etc/secrets/initrd`). If someone is able to access your hardware, they could pretend to be your server and intercept your input of the encryption passphrase. You could move this somewhere else with a KVM, but an attacker could do a similar attack on a KVM if you don't secure access to it.

#### Storage Arrays

My old arrays were encrypted at the disk level with LUKS and then added to a ZFS array. To support them, I needed a way to open LUKS devices and mount the corresponding ZFS arrays. I'm not sure if this is the most efficient way to do this, but I put together a few functions to handle this:

```nix
uuidToDevice = uuid: "/dev/disk/by-uuid/${uuid}";
# Note that the keys are located where they're seen during the init process,
# not where the filesystem is after boot has completed
# See boot.initrd.postDeviceCommands for how the key is handled
keyMount = "/key";
storageKey = "${keyMount}/storage";

storageUUIDs = [
  # Main disks
  ...
];

storageVolumes = (map
  (uuid: {
    name = uuid;
    value = {
      device = uuidToDevice uuid;
      preLVM = false;
      keyFile = storageKey;
    };
  })
  storageUUIDs);
```

Once those functions are defined, you can set

```nix
boot.initrd.luks.devices = builtins.listToAttrs storageVolumes;
```

and generalize the functions to other arrays to append to `storageVolumes`.

## Sharing

For the whole "network attached" part of NAS, we'll need some sort of sharing. In my old setup, this was a nightmarish manual management of `/etc/samba/smb.conf` that would often break from any change.

Luckily, NixOS gives easy access to the shares I needed via [`services.samba`][nixos-samba] and [`services.nfs`][nixos-nfs].

Your shares really depend on your use-case, but I use Samba for most sharing and NFS for my Proxmox servers. It's outside the scope of this post, but it may be worth noting that NFS is insecure by default and the only real security option is Kerberos, where you effectively delegate security to the clients.

## ZFS Maintenance

Keeping your pools in working order is pretty important if you want them to be reliable. Unfortunately, this is something that I haven't spent a ton of time on. NixOS does offer `services.zfs.autoScrub.*` to automatically scrub pools, but you ideally want to be notified when something goes wrong.

The only way to handle this in NixOS 22.05 seems to be `services.zfs.zed.*`, which would work if I had an email stack set up, but I don't. I currently have [a horrible script to send status to Discord][discord-notify-script], but I'm looking for something better when I get a chance. I have ZFS stats exported via Prometheus with

```nix
services.prometheus = {
  exporters = {
    node = {
      enable = true;
      openFirewall = true;
      enabledCollectors = [
          "zfs"
      ];
    };
  };
};
```

which I thought would work when combined with Grafana's alerting, but only low-level stats seem to be exported rather than high-level information like faulted disks.

[mdlayher/zedhook][zedhook] also looks like a possible option, but I haven't dug into it yet.

## Backups

If you care about your data, you should have backups. Personally I use [restic][restic] via `services.restic.backups` and handle secrets with [agenix][agenix], but NixOS also supports other common tools such as borg and rsnapshot.

A robust backup strategy could take an entire post of its own, but you should be sure to test that your backups work and limit the ability for a given host to destroy its own backups in the case of malware.

## Running Applications

A bunch of storage isn't much if you don't do something with that storage! Aside from using network shares on my desktops and servers, I also use it for storage and management of media, such as family photos and my music collection.

My family photos have always been a bit of a mess, but I wanted to get them into order with my NAS rework (in fact, I restored my NAS to access those photos). After looking around at a number of options, it seemed that [PhotoPrism][photoprism] was a solid place to start; it would import all my photos into a database and organize the directory structure. It has a number of interesting features, but it also organizes my photos into a structure that I can migrate to some other application (or build my own!) if I decide to.

NixOS doesn't have a module for PhotoPrism. I considered writing a module for it, but I decided to go a simpler route and just run a container (in the past, most of my NAS applications have been container based). I wrote my own module for this:

```nix
{ config, lib, ... }:
let
  cfg = config.services.photoprism;
in
{
  options.services.photoprism = {
    uid = lib.mkOption {
      type = lib.types.int;
      default = 990;
      description = "UID of protoprism user";
    };

    gid = lib.mkOption {
      type = lib.types.int;
      default = 990;
      description = "GID of photoprism primary group";
    };

    extraGroups = lib.mkOption {
      type = lib.types.listOf lib.types.str;
      default = [ ];
      description = "Additional groups to add to photoprism user";
    };
  };

  config = {
    users.groups.photoprism = {
      gid = cfg.gid;
    };

    users.users.photoprism = {
      isSystemUser = true;
      group = "photoprism";
      extraGroups = cfg.extraGroups;
      uid = cfg.uid;
    };

    virtualisation.oci-containers = {
      backend = "podman";
      containers = {
        photoprism = {
          image = "photoprism/photoprism:latest";
          ports = [ "127.0.0.1:2342:2342" ];
          user = "990:990";
          environment = {
            PHOTOPRISM_ADMIN_PASSWORD = "supersecure";
            PHOTOPRISM_HTTP_COMPRESSION = "gzip";
            PHOTOPRISM_PUBLIC = "false";
            PHOTOPRISM_READONLY = "false";
            PHOTOPRISM_DISABLE_CHOWN = "true";
            PHOTOPRISM_DISABLE_WEBDAV = "true";
            PHOTOPRISM_DATABASE_DRIVER = "sqlite";
            PHOTOPRISM_FFMPEG_ENCODER = "intel";
          };
          volumes = [
            "/dev/dri:/dev/dri"
            "/mnt/media/photos-original:/photoprism/originals"
            "/mnt/media/photos-import:/photoprism/import"
            "/data/photoprism:/photoprism/storage"
          ];
        };
      };
    };
  };
}
```

and used it in `configuration.nix` with

```nix
imports = [
  ./photoprism.nix
];
services.photoprism.extraGroups = [ "your-groups-here" ];
```

## End Thoughts

This was quite a journey and there are still some open questions on properly maintaining it in the future.

I think it would be neat to build up a reusable NAS configuration and make it public in a way that it can just be imported and enabled with something like

```nix
services.nas = {
  enable = true;
  # Shares to configure with sensible defaults for Samba/NFS
  shares = {
    photos = {
      path = "/data/photos";
      nfs.enable = false;
    };
    isos = {
      path = "/data/isos";
      # By default, NFS is enables and the path is
      # /export/${share}
      nfs.readonly = true;
    };
  }
};
```

but that would require a fair bit of refactoring from my system-specific configuration. I may also just open source my specific configurations if I have time to extract out any information that I don't want public.

[agenix]: https://github.com/ryantm/agenix
[discord-notify-script]: https://github.com/danielunderwood/scripts/blob/3b13126ee6146336f68ff7473781f1347d3944bc/zfs-notify.sh
[morph]: https://github.com/DBCDK/morph
[nixos-nfs]: https://github.com/NixOS/nixpkgs/blob/nixos-22.05/nixos/modules/services/network-filesystems/nfsd.nix
[nixos-samba]: https://github.com/NixOS/nixpkgs/blob/nixos-22.05/nixos/modules/services/network-filesystems/samba.nix
[nixos-zfs]: https://nixos.wiki/wiki/ZFS#Pool_layout_considerations
[photoprism]: https://photoprism.app/
[pikvm]: https://pikvm.org/
[restic]: https://restic.net/
[zedhook]: https://github.com/mdlayher/zedhook

[^remote-decryption]: https://mth.st/blog/nixos-initrd-ssh/
[^nixos-wiki-remote-decryption]: https://nixos.wiki/wiki/ZFS#Unlock_encrypted_zfs_via_ssh_on_boot
