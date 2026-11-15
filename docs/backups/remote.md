---
title: Back up storage data remotely
toc: true
---

We want to configure a daily automated incremental remote backup of the data stored on the external master disk, to minimize data loss in case of failure.

We want to ensure it happens in a secure way. We will therefore make use of:

- A dedicated backup user `remote-backup`, in order to isolate backup operations to a passwordless system user on the remote machine.
- `rsync` over SSH, using a system-owned (`root`) Ed25519 SSH key pair (`/etc/ssh/ssh_minus_backup_ed25519`) rather than personal user keys or legacy RSA.

We will automate the remote backup execution using `cron`.

## Assign a permanent IP address to the remote machine

Machines connected to our local network receive their IP address from our main host dynamically, using DHCP. This is thanks to our [dnsmasq](../networking/dhcp-dns.html) configuration.

Unlike other machines, we want our remote backup host to be configured with an IP address permanently assigned to it: `10.0.0.254`.

To achieve this, let's edit our dnsmasq network configuration file using:

```sh
sudo nano /etc/dnsmasq.d/home.lan.conf
```

And add the following line after the `dhcp-range` configuration option:

```sh
# Static IP addresses
dhcp-host=[MAC_ADDRESS],minus,10.0.0.254
```

Where
- `[MAC_ADDRESS]` is the network interface physical address of your remote machine, in the form of `01:23:45:67:89:ab`.
- `minus` is the hostname we want assigned to it.

Apply the new configuration by executing the following:

```sh
sudo systemctl restart dnsmasq
```

## Prepare the remote backup machine

Log into your remote backup machine (in our case: `10.0.0.254` or `minus`) using an existing administrative account to create the backup directory and the dedicated `remote-backup` user.

1. **Create the target backup directory:**
   *(No `chown` is needed here because `rsync` will execute with root privileges on the destination).*
   ```sh
   sudo mkdir -p /mnt/data/backup
   ```

2. **Create the dedicated passwordless backup user:**
   ```sh
   sudo useradd -m -s /bin/bash remote-backup
   ```

3. **Grant passwordless `sudo` specifically for `rsync`:**
   Open a dedicated sudoers configuration file:
   ```sh
   sudo visudo -f /etc/sudoers.d/remote-backup
   ```
   Add the following line and save:
   ```text
   remote-backup ALL=(root) NOPASSWD: /usr/bin/rsync
   ```

Verify that this setup works as expected, by executing the following command to ensure the rule works without prompting for a password:

```sh
sudo -u remote-backup sudo -n /usr/bin/rsync --version
```

## Generate & deploy the system-owned SSH key

On your **source machine**, generate an Ed25519 key pair owned by `root` so the cron job can run unattended without relying on a personal user account.

1. **Open a root shell and generate the key:**
   *(Leave the passphrase empty when prompted).*
   ```sh
   sudo -s
   ssh-keygen -t ed25519 -C "minus-backup-automation" -f /etc/ssh/ssh_minus_backup_ed25519
   ```

2. **Deploy the public key to the remote machine:**
   Because `remote-backup` does not have a password, `ssh-copy-id` cannot be used. Instead, pipe the public key through your existing administrative SSH user (replace `your_admin_user` with your actual admin username on `10.0.0.254`):
   ```sh
   ssh your_admin_user@10.0.0.254 "sudo mkdir -p /home/remote-backup/.ssh && sudo tee -a /home/remote-backup/.ssh/authorized_keys" < /etc/ssh/ssh_minus_backup_ed25519.pub
   ```

3. **Set strict permissions on the remote `.ssh` directory:**
   SSH will reject keys if directory or file permissions are too permissive:
   ```sh
   ssh your_admin_user@10.0.0.254 "sudo chown -R remote-backup:remote-backup /home/remote-backup/.ssh && sudo chmod 700 /home/remote-backup/.ssh && sudo chmod 600 /home/remote-backup/.ssh/authorized_keys"
   ```

4. **Exit the root shell:**
   ```sh
   exit
   ```

*Verification:* 
1. Confirm the local private key is owned by `root` with `600` permissions:
   ```sh
   sudo ls -l /etc/ssh/ssh_minus_backup_ed25519*
   ```
2. Test passwordless SSH and remote `sudo rsync` execution from the source machine (note that `sudo` is required locally to read the root-owned private key):
   ```sh
   sudo ssh -i /etc/ssh/ssh_minus_backup_ed25519 remote-backup@10.0.0.254 "sudo rsync --version"
   ```

## Configure the rsync backup command

We want to ensure that:
- **Exact file/directory ownership & permissions are retained:** We will use `--numeric-ids` and targeted passwordless `sudo rsync` on the remote end to preserve exact file ownership (including `root`-owned files) without requiring `chown` on the backup directory.
- **Optimized for local high-speed network:** We will omit the `-z` compression flag to maximize throughput and avoid unnecessary CPU overhead over fast local networks.

### Perform a dry run first

Before automating the job, let's verify that paths, permissions, and file comparisons behave as expected, by performing a dry-run execution.

```sh
sudo /usr/bin/rsync -av --dry-run --numeric-ids -e "sudo ssh -i /etc/ssh/ssh_minus_backup_ed25519" --rsync-path="sudo rsync" --delete /mnt/data/master/ remote-backup@10.0.0.254:/mnt/data/backup/
```

*Note: Ensure the trailing slashes `/` on both `/mnt/data/master/` and `/mnt/data/backup/` are present so contents are synced directly rather than creating nested subdirectories.*

Immediately run `echo $?` after the command finishes. An exit code of `0` confirms a clean execution.

### Run an actual sync

Once the dry run confirms only expected changes will occur, remove `--dry-run` to execute a live sync:

```sh
sudo /usr/bin/rsync -av --numeric-ids -e "sudo ssh -i /etc/ssh/ssh_minus_backup_ed25519" --rsync-path="sudo rsync" --delete /mnt/data/master/ remote-backup@10.0.0.254:/mnt/data/backup/
```

Immediately run `echo $?` after the command finishes. An exit code of `0` confirms a clean execution.

## Automate execution

We will automate the execution of this remote backup via system crontab (`/etc/crontab`). We will schedule it to run daily at a low activity time: 2:00 in the morning.

Let's edit `/etc/crontab`:
```sh
sudo nano /etc/crontab
```

And add the scheduled entry at the bottom of the file:
```sh
00 2    * * *   root    /opt/backup/data-remote.sh 1> /dev/null 2> /dev/null
```

*Remember that `/etc/crontab` requires specifying the execution user `root` before the command.*

Once added, save and close the file.

You do not need to restart or reload the `cron` daemon; it automatically detects changes to `/etc/crontab` every minute.
