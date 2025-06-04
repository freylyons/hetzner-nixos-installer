# Recovery Process
If you happen to be an idiot and make a change like disabling `networking.useDHCP` which was set as default in your `hardware-configuration.nix` file, then you might be in need of a recovery method for your nixos server. But how do you do that? Below is a simple recipe for unfucking the fuck stick.

## 1. Boot into the Rescue System and Mount your block devices

You must first boot into the rescue system before you can make any fixes (don't forget to ssh root@x.x.x.x instead of your normal username for hetzner). Mount your block devices and create a nix-shell environment, although you don't need to install any packages for this one - just having nix will do.

## 2. Inspect your generations

Not to worry, since nixos supports atomic rollbacks by default, we just need to find the previous generation we were working on and tell nixos to boot it.

We inspect our generations with
```
sudo nix-env --list-generations --profile /mnt/nix/var/nix/profiles/system
```
and get something like 
```
WARNING: terminal is not fully functional
Press RETURN to continue 
   1   2025-04-27 16:35:41   
   2   2025-04-27 16:37:56   
   3   2025-04-27 18:01:00   
   4   2025-04-27 18:01:27   
   5   2025-04-27 23:35:22   
   6   2025-04-27 23:40:07   
   7   2025-04-27 23:41:39   
   8   2025-04-28 00:11:55   
   9   2025-04-29 19:11:22   
  10   2025-04-29 19:12:23   
  11   2025-05-18 13:19:39   
  12   2025-05-18 15:09:38   
  13   2025-05-18 15:21:20   
  14   2025-05-18 21:30:28   
  15   2025-06-04 20:58:56   
  16   2025-06-04 21:08:28   
  17   2025-06-04 21:13:07   
  18   2025-06-04 21:18:05   
  19   2025-06-04 21:55:46   (current)
```

very simple! This will show us our current generation (the one you fucked) and the other ones the bootloader can access.

Now we choose a previous generation that we want to restore from (in my case, 17, because I tried to install nixos again which made a new generation that didn't work)
```
nix-env --switch-generation 17 --profile /mnt/nix/var/nix/profiles/system
```

which will output something like ```switching profile from version 19 to 17```

After than, you can safely `reboot`!

Your unfucked nixos will be available to ssh into shortly :)


