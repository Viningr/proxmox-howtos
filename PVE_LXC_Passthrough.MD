# Nvidia GPU enabled LXC containers on Proxmox 8.3.1.

# Before starting, make sure that the Nvidia driver (and potentially the nvidia-container-toolkit) is installed on the host. Note the driver version and ensure you have configured DKMS on your system if it uses secure boot/uefi. The driver must have kernel modules enabled. Verify the driver is reporting in the nvidia-smi tool. Additional steps may be required to configure passthrough, I know this is needed on VMs which consume the GPU, but containers may not require.

- Create container, ensure you uncheck the unprivileged setting during container creation.
- Once created, under Options\Features, enable Nesting.
- Under Resources, add a passthrough device, typically /dev/nvidia0 on a single GPU system. Use the default settings and apply.

- Start the container to verify basic functionality before setting up GPU permissions and symlinks. Shut the container down once you have confirmed you can log on.

- On your host, find the appropriate permissions to add to the LXC config:

```bash
ls -l /dev/nvidia*

This will return something similar to:

```bash
root@pve:/etc/pve/lxc# ls -l /dev/nvidia*
crw-rw-rw- 1 root root 195,   0 Dec  2 23:10 /dev/nvidia0
crw-rw-rw- 1 root root 195, 255 Dec  2 23:10 /dev/nvidiactl
crw-rw-rw- 1 root root 195, 254 Dec  2 23:10 /dev/nvidia-modeset
crw-rw-rw- 1 root root 507,   0 Dec  2 23:10 /dev/nvidia-uvm
crw-rw-rw- 1 root root 507,   1 Dec  2 23:10 /dev/nvidia-uvm-tools

/dev/nvidia-caps:
total 0
cr-------- 1 root root 510, 1 Dec  2 23:10 nvidia-cap1
cr--r--r-- 1 root root 510, 2 Dec  2 23:10 nvidia-cap2

- From this output you can determine the cgroup2 permissions:

### Device permissions for NVIDIA GPU passthrough
lxc.cgroup2.devices.allow: c 195:* rwm  # For nvidia0, nvidiactl, nvidia-modeset
lxc.cgroup2.devices.allow: c 507:* rwm  # For nvidia-uvm, nvidia-uvm-tools
lxc.cgroup2.devices.allow: c 510:* rwm  # For nvidia-cap1, nvidia-cap2

### Mount entries for passthrough
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap1 dev/nvidia-caps/nvidia-cap1 none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap2 dev/nvidia-caps/nvidia-cap2 none bind,optional,create=file

- add the "lxc.autodev: 0" property to the config as well, as we are manually configuring the device.

## Example ctid.conf additions:
<existing CTID.conf contents>...
lxc.autodev: 0
lxc.cgroup2.devices.allow: c 195:* rwm  # For nvidia0, nvidiactl, nvidia-modeset
lxc.cgroup2.devices.allow: c 507:* rwm  # For nvidia-uvm, nvidia-uvm-tools
lxc.cgroup2.devices.allow: c 510:* rwm  # For nvidia-cap1, nvidia-cap2
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap1 dev/nvidia-caps/nvidia-cap1 none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap2 dev/nvidia-caps/nvidia-cap2 none bind,optional,create=file


- Once you have added the permissions and mount entries, you will need to start the container again, then log on and install the Nvidia driver. Ensure this is the same version installed on the host. When you install (use the .run version) use the `--no-kernel-module` flag after the driver run file name.

- Once complete, run the nvidia-smi tool inside the container and confirm it runs and reports the driver version.

# Adding CUDA support

- Sign into the container.
- Update Apt
apt update

- Install the Nvidia keyring. Navigate to an appropriate folder then run:
wget https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64/cuda-keyring_1.1-1_all.deb
dpkg -i cuda-keyring_1.1-1_all.deb

- Add the CUDA repository:
echo "deb [signed-by=/usr/share/keyrings/cuda-archive-keyring.gpg] https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64/ /" | sudo tee /etc/apt/sources.list.d/cuda.list
apt update

### (switch: choose one based on your requirements)

- Full CUDA toolkit (assuming cuda 12.4)
apt -y install cuda-toolkit-12-4 

- Tailored packages for smaller footprint:
apt install cuda-cudart-12-4 cuda-nvcc-12-4 #CUDA Runtime library and CUDA compiler

### Add enviroment variables to the shell:

- Append these lines to your .bashrc file
echo 'export PATH=/usr/local/cuda-12.4/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12.4/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
echo 'export CUDA_HOME=/usr/local/cuda-12.4' >> ~/.bashrc

- Reload the .bashrc file
source ~/.bashrc



## An example of a systemd service configuration which includes the CUDA environment declarations:

[Unit]
Description=CUDA-Enabled Application Service
After=network.target

[Service]
# Type of service - here, we assume it's a simple foreground process
Type=simple

# User and group to run the service as (modify according to your setup)
User=cudauser
Group=cudagroup

# Set the environment variables for CUDA
Environment=PATH=/usr/local/cuda-12.4/bin:$PATH
Environment=LD_LIBRARY_PATH=/usr/local/cuda-12.4/lib64:$LD_LIBRARY_PATH
Environment=CUDA_HOME=/usr/local/cuda-12.4

# If you need additional environment variables, you can use EnvironmentFile instead
# EnvironmentFile=/etc/sysconfig/cuda_env

# Command to execute your CUDA application
ExecStart=/usr/bin/your_cuda_application

# If your application forks, you might want to use:
# Type=forking
# PIDFile=/var/run/your_cuda_application.pid

# Restart policy
Restart=on-failure
RestartSec=10

# Optional: Standard output and error logging
StandardOutput=journal
StandardError=inherit

# Resource limits if needed (example)
# LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
