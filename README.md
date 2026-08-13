# docker-lab
Home lab for deploying and administering PostgreSQL with Docker on Ubuntu Server.

## Goals

- Deploy an Ubuntu Server virtual machine
- Configure remote administration over SSH
- Install and configure Docker
- Deploy PostgreSQL in a container
- Configure persistent storage
- Configure container networking
- Test PostgreSQL connectivity
- Document troubleshooting and configuration decisions

## Environment

| Component | Configuration |
|-|-|
| Host | MacBook Pro M1 Pro |
| Hypervisor | UTM |
| Guest OS | Ubuntu Server 24.04.4 LTS |
| Architecture | ARM64 / aarch64 |
| VM CPU | 4 vCPU |
| VM RAM | 4 GB |
| VM Disk | 40 GB |
| Storage | LVM + ext4 |
| Remote access | OpenSSH |

## Architecture

```text
MacBook
   |
   | SSH
   |
   v
Ubuntu Server VM
   |
   v
Docker Engine
   |
   v
PostgreSQL container
```

## 1. Ubuntu Server Setup

Ubuntu Server 24.04.4 LTS was deployed as an ARM64 virtual machine in UTM.

### System verification

```bash
cat /etc/os-release
uname -m
ip addr
df -h
free -h
```

The VM is running on the `aarch64` architecture and receives its initial IPv4 address via DHCP.

### Storage configuration

Ubuntu was installed using LVM. The default installation allocated approximately half of the volume group to the root logical volume.

The root logical volume was extended to use all available space:

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

The resulting LVM configuration was verified with:

```bash
sudo pvs
sudo vgs
sudo lvs
df -h /
```

### SSH access

OpenSSH Server was installed during Ubuntu setup.

Avahi was installed to make the VM accessible by its hostname via mDNS:

```bash
sudo apt update
sudo apt install -y avahi-daemon
```

The VM can then be accessed from the macOS host using:

```bash
ssh ruslan@docker-lab.local
```
## 2. Docker installation

### Set up Docker's apt repository

```apt``` prerequisites and Docker GPG key setup:

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Adding the repository to ```apt``` sources:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

```bash
sudo apt update
```

### Install the Docker packages

Installing the latest version:

```bash
sudo apt install docker-ce docker-ce-cli containerd.io \
docker-buildx-plugin docker-compose-plugin
```

Docker service status check:

```bash
sudo systemctl status docker
```

### Smoke test

Running the test container:

```bash
sudo docker run hello-world
```

Successful launch!

## 3. PostgreSQL deployment

Installing the latest PostgreSQL version:

```bash
sudo apt install postgresql
```

PostgreSQL service status check:

```bash
sudo systemctl status postgresql
```

Smoke test:

```bash
sudo -u postgres psql -c "SELECT version();"
```

Succesful launch too!

## 4. Networking

To be completed.

## 5. Persistent storage

To be completed.

## 6. Testing

To be completed.

## Troubleshooting

### ```sudo apt update``` error

After adding the ```Docker``` repository an error occurred:

```bash
sudo apt update
E: Malformed entry 1 in sources file /etc/apt/sources.list.d/docker.sources (URI)
E: The list of sources could not be read.
```

```docker.sources``` was checked:

```bash
cat /etc/apt/sources.list.d/docker.sources

Types: deb

URIs: https://download.docker.com/linux/ubuntu

Suites: noble

Components: stable

Architectures: arm64

Signed-By: /etc/apt/keyrings/docker.asc

```

The file contained an empty line after each entry, causing ```apt``` to parse them as separate entries.

```bash
sudo nano /etc/apt/sources.list.d/docker.sources
```

After removing the empty lines, the file looked like this:

```bash
ruslan@docker-lab:~$ cat /etc/apt/sources.list.d/docker.sources
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: noble
Components: stable
Architectures: arm64
Signed-By: /etc/apt/keyrings/docker.asc
```

After the fix, apt update completed successfully.
