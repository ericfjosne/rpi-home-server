---
title: Set up encryption on root partition
---

By default, the Raspberry Pi OS root partition is not encrypted. To encrypt it using LUKS, [follow this guide](https://sindro.me/posts/2026-01-20-raspberry-pi-luks-encrypted-root/).

After you are done with all the steps, if you want to automatically be prompted for the passphrase to unlock the root partition at boot, you will need `cryptsetup-initramfs` by executing the following:

```sh
sudo apt install cryptsetup-initramfs
```

Once installed, update `initramfs` by executing the following:

```sh
sudo update-initramfs -u
```

Next time you reboot, you will be able to unlock your disk immediately.
