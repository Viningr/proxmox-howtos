# Set up xtts-api-server in PVE LXC container.

##This is my documentation of setting up xtts-api-server as a systemd service on a Proxmox hosted LXC container with GPU passthrough. The xtts-api-server project is authored by daswer123 at https://github.com/daswer123/xtts-api-server .


##1. Set up the container and container prerequisites:

- Create a new LXC container based on the debian12-nvidia template created earlier.

## Initial Specs:
* unprivileged: 0
* cores: 4+
* memory: 8192
* static IPv4
* storage: 20GB

- Complete provisioning of LXC.
- Update Resources with addition of passthrough device /dev/nvidia0.
- Update Options with addition of Nesting under Features.
- Start LXC and confirm it boots and shell is accessible.
- Shutdown LXC.
- Log into PVE host shell.
- Edit the LXC configuration.
```bash
nano /etc/pve/lxc/<CTID>.configuration
```

- Append prederived permissions and symlinks and save\exit:

```ini
***<Existing <CTID>.conf properties here...***
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
```
- Start the LXC and log into the shell.
- Test passthrough is working, the following command should show the nvidia driver and cuda versions, if this fails, review your steps and retry:
```bash
nvidia-smi
```
- Changed to ~/nvidia/ (make this folder if it does not exist) and ran the following to install the keychain:
```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64/cuda-keyring_1.1-1_all.deb
dpkg -i cuda-keyring_1.1-1_all.deb
```

- Add CUDA repo to sources and update apt.
```bash
echo "deb [signed-by=/usr/share/keyrings/cuda-archive-keyring.gpg] https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64/ /" | tee /etc/apt/sources.list.d/cuda.list
apt update
```

- Install the CUDA runtime.
```bash
apt install cuda-cudart-12-4 #CUDA Runtime library
```
*** NOTE: Deepspeed flag is not supported when running this application from systemd services. Project needs to be updated with support for precompiled wheel in linux. ***

- Add the environment variables to the shell:
```bash
echo 'export PATH=/usr/local/cuda-12.4/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12.4/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
echo 'export CUDA_HOME=/usr/local/cuda-12.4' >> ~/.bashrc
```

- Install Python, venv, pip, git, ffmpeg and portaudio19-dev.
```bash
apt update
apt install python3-pip python3-venv git portaudio19-dev ffmpeg -y
```

- Activate the newly updated bash environment.
```bash
source ~/.bashrc
```

## 2. Install the xtts-api-server project.

- Pick a location to host the project files, I will be installing to ~/ (/root/). Please be aware this is poor practise if you intend to put this into production, these values are for demonstration purposes only.
```bash
cd ~
```

- Follow the manual installation process from the project repo:
```ini
# Clone REPO
git clone https://github.com/daswer123/xtts-api-server
cd xtts-api-server
# Create virtual env
python3 -m venv venv
# Add the CUDA environment variables to the venv (addition to the original install process):
cat << EOF >> ~/xtts-api-server/venv/bin/activate
export PATH="/usr/local/cuda-12.4/bin:\$PATH"
export LD_LIBRARY_PATH="/usr/local/cuda-12.4/lib64:\$LD_LIBRARY_PATH"
export CUDA_HOME="/usr/local/cuda-12.4"
EOF
# Activate the venv:
source venv/bin/activate #(confirm you see the prompt change).
# Install deps
pip install -r requirements.txt

# Launch server
python -m xtts_api_server --listen --device cuda:0 -v v2.0.3
```

- Confirm the program loads and is connectable on the container IP port 8020.

- Once the manual validation is successful, transfer your voice sample files to ~/xtts-api-server/speakers.

- Create a new systemd service.
```bash
nano /etc/systemd/system/xtts-api-server.service
```

```ini
[Unit]
Description=XTTS-API-Server Service
After=network.target

[Service]
Type=simple
User=root
Group=root
WorkingDirectory=/root/xtts-api-server
Environment="PATH=/root/xtts-api-server/venv/bin:/usr/local/cuda-12.4/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="LD_LIBRARY_PATH=/usr/local/cuda-12.4/lib64"
Environment="CUDA_HOME=/usr/local/cuda-12.4"
ExecStart=/root/xtts-api-server/venv/bin/python -m xtts_api_server --listen --device cuda:0 -v v2.0.3
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```
- Enable the new service:

```bash
systemctl daemon-reload
systemctl enable xtts-api-server
systemctl start xtts-api-server
```

- Test the service. It should be reachable on the container IP:8020 over http. Refer to the original project repository for more information on program operation and arguments. You can check the status with:
```bash
systemctl status xtts-api-server
```

- ***The container will automatically start the xtts-api-server on boot.***

## The final container can be optimized after setup, depending on how much space you use for voice samples, there should be 25-30% of the 20GB boot disk free. Final CT specs that work well for me:
```ini
vCPU: 2
RAM: 4096 MB
SWAP: 512 MB
Host GPU vram: 3784 MB used after inference.
```
