# Install Notes

### sources
https://github.com/nix-community/nixos-images#kexec-tarballs
https://wiki.nixos.org/wiki/Install_NixOS_on_Hetzner_Online
https://nixos.org/manual/nixos/stable/#sec-installation-manual

## Prerequisites

- The hetzner server must be in rescue mode with your public key activated for ssh. Without the public key the password will change after step 1 and you will be locked out.

## First steps - download bootstrap
visit `https://github.com/nix-community/nixos-images#kexec-tarballs` and execute:

```
curl -L https://github.com/nix-community/nixos-images/releases/latest/download/nixos-kexec-installer-noninteractive-x86_64-linux.tar.gz | tar -xzf- -C /root
/root/kexec/run
```

This installs the bootstrapper. Wait 6 seconds for the kexec to start and then kill the shell

## Step 2: Disk Formatting

follow the instructions from
- https://nixos.org/manual/nixos/stable/#sec-installation-manual-partitioning-UEFI
- https://nixos.org/manual/nixos/stable/#sec-installation-manual-partitioning-formatting

## Step 3. Installation

follow the instructions from
- https://nixos.org/manual/nixos/stable/#sec-installation-manual-partitioning-formatting
until 5. (exclusive)

install an editor
```
nix shell neovim
```
edit /mnt/etc/nixos/configuration.nix

paste in the following from https://wiki.nixos.org/wiki/Install_NixOS_on_Hetzner_Online

```
boot.loader.grub.device = "/dev/nvme0n1";
services.openssh.enable = true;
users.users.root.openssh.authorizedKeys.keys = [
  "<ssh-public-key>"
];
```
You need to do this to ensure the bootloader works and that you can access the new install with ssh. Make sure that the bootloader entry here is for the same bootloader as the generate one, or the one you want to use.

Add a version of nixpackages to the nix path, e.g.
```
export NIX_PATH=nixpkgs=https://github.com/NixOS/nixpkgs/archive/nixos-23.11.tar.gz
```

Finally run the installer
```
nixos-install
```

# Round 2

## Upload the iso to the server

Mount/swapon the block devices as per the installation instructions

Upload the minimal nixos iso to `/mnt/root/nixos-minimal-24.11.717296.5630cf13ccea-x86_64-linux.iso`

```
mkdir -p /mnt/iso && mount -o loop /mnt/root/nixos-minimal-24.11.717296.5630cf13ccea-x86_64-linux.iso /mnt/iso
```

install nix and start a nix shell with kexec-tools
```
sh <(curl -L https://nixos.org/nix/install) --daemon
bash
nix-shell -p kexec-tools
```
populate the new system with your public ssh key
```
mkdir /tmp/newroot
cp -a /mnt/iso/* /tmp/newroot/

mkdir -p /tmp/newroot/root/.ssh
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILS1Gr8OBAemjzv4waPFvNpjXUOAYpIBLZlBiQyrnukO frey@Freys-PC" > /tmp/newroot/root/.ssh/authorized_keys
```


generate the kexec and jump to the new system
```
ls /mnt/iso/boot/

# load the kernel
kexec -l /tmp/newroot/boot/bzImage --initrd=/tmp/newroot/boot/initrd --command-line="systemd.unit=emergency.target systemd.network.useDHCP=false ip=:188.40.98.158:188.40.98.129:255.255.255.0::eth0:none"

# jump into the new system 
kexec -e
```

TODO on return:
- enable rescue system
- use `find /mnt/iso -name init`
- put the output of that into
```
--command-line="boot=live systemd.network.useDHCP=false ip=:188.40.98.158:188.40.98.1:255.255.255.0::eth0:none"
``` 
(this is the ip I access with ssh and the gateway hopefully should be correct)
- double check the network device name before committing
https://chatgpt.com/c/680d24c9-8c70-8012-9649-0db90a6f4b0e

# Attempt 3

## mount the drives
```
mount /dev/disk/by-label/nixos /mnt
mount -o umask=077 /dev/disk/by-label/boot /mnt/boot
swapon /dev/nvme1n1p2
```

## install nix and build packages
```
sh <(curl -L https://nixos.org/nix/install) --daemon
bash
nix-shell -p neovim nixos-install-tools parted
```

## edit config to configure bootloader and add ssh public key
`nvim /mnt/etc/nixos/configuration.nix`
```
# Use the systemd-boot bootloader
{
  boot.loader.systemd-boot.enable = true;
  boot.loader.systemd-boot.configurationLimit = 10; # set maximum number of bootloader entries to prevent bootloader lag
  
  users.users.root.openssh.authorizedKeys.keys = [
    "<ssh-public-key>"
  ];
}
```

## install nixos
```
nixos-install --root /mnt
```
