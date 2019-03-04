# Machine Learning Infrastructure to-go
Updated: Mar 4, 2019
(c) 2019 by Ray Lai

## Summary
This document explains how to build a compact Machine Learning infrastructure using Docker and Kubernetes for small scale Machine Learning development use.  It is best suitable for early startup during the bootstrap phase.

The features include:
* Setting up a GPU server
  - CUDA 10.x driver integration with Ubuntu 18.x
  - Non-root user and RBAC consideration
  - Jupyter notebook with GPU integration
  - Virtualization options, e.g. docker, kubernetes
* Network security infrastructure
  - "Private cloud in a box"
  - Use of VPN and MFA
  - Anti-virus and malware detection
  - Data security options, e.g. host-based intrusion detection
* Kubernetes cluster
* Machine Learning workflow orchestration
  - Build pipeline for Machine Learning
  - Deployment pipeline for Machine Learning


## GPU Server Installation Guide
* Bare Metal Server Provisioning
You have an entry level GPU server (e.g. Intel Core i7 CPU with 128GB memory with a single Nvidia GTX 2080Ti GPU), you want to set up the Linux Operating System, GPU driver and the Python/Jupyter notebook development environment for data scientists. You may have some basic desktop PC or server knowledge of how to assemble a desktop server.

### High Level Steps
1. Pre-requisites: (1) Assemble GPU server - install GPU, connect monitor and keyboard/mouse to the server. (2) Prepare a bootable USB thumb drive with the target OS, e.g. use BalenaEtcher to create a bootable OS image using Ubuntu 18 ISO image (https://www.balena.io/etcher/).
2. Install Ubuntu.  Insert the bootable USB thumb drive.  Boot and start the GPU server. Make sure you select the boot sequence to use the USB thumb drive from the server BIOS menu.
3. Install Nvidia CUDA driver.  Remove or clean up any previous CUDA driver first.
4. Install docker and nvidia-docker.
5. Deploy Jupyter notebook docker image.

### Recommended Server Provisioning Configuration
Most Machine Learning applications require latest Linux drivers and packages. Ubuntu is commonly used due to the availability of latest Linux drivers and techical documentation of the ML framework integration. I recommend using Ubuntu.

* Create an admin user, e.g. myadmin
```
sudo mkdir /home/myadmin
sudo useradd -g staff -s /bin/bash myadmin
sudo chown myadmin:staff /home/myadmin
```

* Enable sudo access
If you want to enable this admin user with sudo access:
```
sudo vim /etc/sudoers
```

Append this line to the end of the file:
```
myadmin       ALL=(ALL:ALL) NOPASSWD: ALL
```

* Allow SSH access for backend production support


This assumes your /etc/ssh/sshd_config to enable the SSH key authentication option. Refer to https://www.ssh.com/ssh/sshd_config/ for details, e.g. the following sshd_config options will allow using only SSH keys to access the server, and can remotely open a Web browser from the server via X11.
```
X11Forwarding yes
PubkeyAuthentication yes
```

### Linux Packages
For most Machine Learning packages, you may want to install these Ubuntu packages:
```
sudo apt-get install -y openssh-server
sudo apt-get -y install ecryptfs-utils
sudo apt-get install -y nfs-common
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt-get install nvidia-driver-415
```

## Network Security Infrastructure
### Private Cloud in a Box
For on-premise deployment, you may want to create a private cloud with VPN and MFA to secure network access.

* VPN options - for novice users, you can create a VPN connection, e.g. Cisco Meraki security appliance allows you to enable users to connect to your corporate network using VPN.  
* VLAN options - In addition to VPN, you can also create a separate VLAN to separate your Machine Learning infrastructure from the remaining network for better isolation.  For example, Cisco Meraki allows you to create a separate VLAN, and assign specific host IP addresses to the VLAN.
* MFA option - You can enable MFA on Linux, e.g. https://www.techradar.com/how-to/how-to-add-two-factor-authentication-to-linux-with-google-authenticator

 
## Docker

```
sudo apt-get update
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
curl https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```
 
You also need to make post-installation changes to auto-start Docker after system reboot:
* Enable docker to be started at boot time
```
sudo systemctl enable docker
sudo systemctl start docker.service
sudo usermod -G docker myadmin
```

Refer to http://docs.docker.com/install/linux/linux-postinstall for post-installation changes.

### Install nvidia-docker for GPU integration
```
docker volume ls -q -f driver=nvidia-docker | xargs -r -I{} -n1 docker ps -q -a -f volume={} | xargs -r docker rm -f
sudo apt-get purge nvidia-docker
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | \
  sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update
```


## Install Nvidia GPU Drivers (aka CUDA)
The wiki under http://www.pugetsystems.com/labs/hpc/How-To-Install-CUDA-10-together-with-9-2-on-Ubuntu-18-04-with-support-for-NVIDIA-20XX-Turing-GPUs-1236/vidia-smi depicts CUDA 10 on Ubuntu 18.x. Please make sure you have matching CUDA driver version with Ubuntu OS version. I once installed CUDA 9.x and 10.x drivers on Ubuntu 18.x (with data volumes encrypted), and the conflicting CUDA drivers corrupted the GRUB boot loader. The quickest means to resolve the issue that time was to re-install the entire GPU server.

```
sudo apt-get install build-essential dkms
sudo apt-get install freeglut3 freeglut3-dev libxi-dev libxmu-dev
sudo apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/7fa2af80.pub
sudo dpkg -i cuda-repo-ubuntu1804_10.0.130-1_amd64.deb
sudo apt-get update
sudo apt-get install cuda

```

For post-installation verification:
* Reboot the server.
* Verify if nvidia-smi can recognize the GPU.
* Add CUDA driver to your login profile, e.g.

vim ~/.bash_profile and add the contents below:

```
export TERM=xterm-color
export PATH=$PATH:/usr/local/cuda-10.0/bin:.
export CUDADIR=/usr/local/cuda-10.0
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda-10.0/lib64
```

## Install Python3
```
sudo apt-get -y install python3
sudo apt-get -y install python3-pip
pip3 install numpy torch torchvision matplotlib==3.0.2 flask flask-cors scikit-learn keras
```

