# Creating LXC Portable Template from Existing LXC

This is a quick guide from another user's Reddit post at [https://www.reddit.com/r/homelab/comments/5xvfbf/how_to_proxmox_modify_a_ct_container_template/](https://www.reddit.com/r/homelab/comments/5xvfbf/how_to_proxmox_modify_a_ct_container_template/). I have summarized this to provide a clear process for creating privileged GPU passthrough containers for my own work. This guide assumes the standard file paths of a default Proxmox 8.x+ installation.

## Prerequisites

- **You must have full permissions on the Proxmox host; this guide assumes root access to the server.**
- **You must have sufficient storage available to backup your existing LXC.**
- **This process can be destructive to the base LXC due to the removal of system settings. Ensure you are working from a clone if you need to continue using the base LXC. Working from a clone will require additional storage space to accommodate.**

##1. Prepare the LXC

- Log into the LXC.
- Update the LXC (optional):
```bash
apt update && apt upgrade -y
```

- Install any desired packages (optional; Python used as an example):
```bash
apt install python3 python3-pip
```

- Clean up the filesystem and unique machine properties:
```bash
apt clean
apt autoclean
apt autoremove
rm /etc/hostname
rm /etc/resolv.conf # (optional: for some setups, preserving resolv.conf might be necessary if there are specific DNS settings required.)
rm /etc/machine-id
```

- From the PVE host UI, shutdown the LXC container.

- In the PVE host UI, remove any static IPs that may have been configured on the container.

##2. Clone the LXC and Convert the Clone to a Portable Template
- In the PVE UI, clone the target LXC. Once the clone completes, right-click the new instance and convert it to a template.

- Create a new ZST backup with Mode: Stop.

- Log into the PVE shell and find the backup file:
```bash
ls /var/lib/vz/dump
```

- Move the backup to your portable container template cache (this will remove the backup from backups for the container):
```bash
mv /var/lib/vz/dump/vzdump-lxc-<CTID>-<CreationDateTime>.tar.zst /var/lib/vz/template/cache/<descriptive-name-for-template>.tar.zst
```
- (Optional) Clean up the leftover files from the dump folder:
```bash
rm /var/lib/vz/dump/*<CTID>*
```

- (Optional) Delete the original template container after verifying the new container template is working.

##3. Profit!
You will now find the new template under the "Create CT" dropdown menu. In my testing, a template created this way from a privileged container will not start as a new unprivileged container due to the isolations designed in unprivileged LXC. If you make a mistake, you can edit the /etc/pve/lxc/<CTID>.conf file and change the unprivileged: 1 property to 0. Ensure the new containers you are creating match the original containers' privileged/unprivileged state.

###References
Original post at https://www.reddit.com/r/homelab/comments/5xvfbf/how_to_proxmox_modify_a_ct_container_template/

##DISCLAIMER
It should go without saying, if you are working with data that is important, make sure you have a separate backup stored somewhere safe. This process will be moving backup files; do not move or delete the wrong one. While this guide has been written in good faith, I am not responsible for any actions you take in following it. If you do not understand commands or concepts, please review before working on anything valuable.
