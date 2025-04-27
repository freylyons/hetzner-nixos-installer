# Final Install process

As per the other `install_process.md` file, it is obvious that working this out has been a colossal pain in the arse. At every road I have been led down rabbit holes, first with the nixos-anywhere ssh installer, then with the kexec approach used both for that and on the wiki. The actual solution turned out to be a hell of a lot simpler and more closely resembles a standard nixos installation.


## First Observations

There are a few things that need to be considered that are unique to installing to hetzner:
    -   Hetzner used legacy boot (BIOS only -> Grub)
    -   Networking needs to be configured to allow access to and from the machine over the internet
    -   SSH needs to be enabled on the machine so that it can be accessed (ideally with your ssh public key)

Generally supplying a public key makes this whole process a lot easier, so save yourself the hassle and generate a public key if you haven't done so already (look at the docs for ssh and the ssh-keygen command)

## Assumptions

This guide assumes that the user is familiar with manual operating system installation processes (e.g. you can install an arch linux distribution comfortably), and that you have some familiarity with a local nixos installation process. It is also helpful to be familiar with ssh and neovim. 


## Initial Steps

First boot the machine into the rescue system. You will want to provide the rescue system with your public key as it makes the process a hell of a lot easier. If you don't, you will have to save a new password each time you need to reactivate the rescue system when you inevitably fuck up.

Once you have booted the rescue system (takes usually just over a minute), ssh into the system.

From here, you will want to follow the standard nixos manual installation instructions. You can ignore the networking section and continue on to format and mount the disks, taking care to follow the legacy boot (MBR) instructions.
https://nixos.org/manual/nixos/stable/#sec-installation-manual

## Creation of build environment

As we are not in a nixos live environment, we will need to install the tooling necessary to continue further. Luckily, thanks to the genious of nix's design, this is a remarkably straightforward process. We just need to install the nix package manager and create a nix shell with a text editor and `nixos-install-tools`. Please note that nano doesn't work here so I have used neovim. I would recommend just using this because it works well, and other options may require more debugging than would take to learn the basics of the editor, which is very useful for editing in remote servers like this anyway.

We follow the same installation instructions as on `https://nixos.org/download/` for a multi-user installation.
```
sh <(curl -L https://nixos.org/nix/install) --daemon
bash # we need to reload the shell to recognise nix commands
nix-shell -p neovim nixos-install-tools # install our tooling and activate the environment
```

The CLI should visually indicate that you are in the nix-shell instead of the rescue system. The rescue system is still active, but you now have access to your nix packages.

## Configuration.nix

From within the nix-shell, follow the instructions to generate the configuration.nix as per the manual installation instructions.

```
nixos-generate-config --root /mnt
```

This is where the installation process becomes specific to hetzner. Use neovim to edit your configuration.nix
```
nvim /mnt/etc/nixos/configuration.nix
```

We will follow the instructions from the nixos wiki for hetzner now.
https://wiki.nixos.org/wiki/Install_NixOS_on_Hetzner_Online

### Networking

First, copy/paste the necessary networking configuration, whether you want to use IPV6 or IPV4/IPV6. Make sure to change the values for your ip addresses, network controllers, and default gateways. You can check these with:
```
ip a # shows ipv4 controller (eth0 or enp...) and address as inet (ignore "lo")
ip -6 a # shows ipv6 controller (eth0 or enp...) and address as inet6 (ignore "lo")
ip route # shows ipv4 default gateway as "default via"
ip -6 route # shows ipv6 default gateway as "default"
```

### Bootloader

Next, ensure that the grub bootloader line is uncommented and that the `bootloader.grub.device` line is uncommented and references the first partition of the disk you used. You need to use grub instead of systemd because systemd does not support BIOS.
```
boot.loader.grub.enable = true;
boot.loader.grub.device = "/dev/nvme0n1";
```

### SSH

Finally, enable the openSSH line and paste in the following:
```
services.openssh.enable = true;
users.users.root.openssh.authorizedKeys.keys = [
  "<your-public-key-here>"
];
```

This both ensures that SSH is enabled in your installation and passes in the public key so that you don't need to use password authentication.


Save the file and close.

## Installation

Finally, if all goes well we can finally install the operating system! Simply run
```
nixos-install --root /mnt
```

Allow the installer to run and reboot when done! It should take about 30 seconds to a minute to boot at which point you can ssh into it. Pay particular attention to errors concerning the bootloader during installation as this will provide an indication if something has gone wrong.
