# docker-lab
Home lab for deploying and administering PostgreSQL with Docker on Ubuntu Server.

## Goals

- Deploy an Ubuntu Server virtual machine
- Configure remote administration over SSH
- Install and configure Docker
- Deploy and verify PostgreSQL as a system service
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
MacBook Pro M1 Pro
       |
       | SSH
       v
Ubuntu Server VM
├── Docker Engine
└── PostgreSQL service
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

The default SSH port was changed from 22 to 2007 in:

```bash
sudo nano /etc/ssh/sshd_config
```

The following directive was set:

```text
Port 2007
```

The VM can then be accessed from the macOS host using:

```bash
ssh -p 2007 ruslan@docker-lab.local
```

The listening port can be verified with:

```bash
ss -tlnp | grep 2007
```

## 2. Docker installation

### Set up Docker's apt repository

`apt` prerequisites and Docker GPG key setup:

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Adding the repository to `apt` sources:

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

The Docker installation was verified successfully.

## 3. PostgreSQL deployment

Installing PostgreSQL from Ubuntu repositories:

```bash
sudo apt install postgresql
```

PostgreSQL service status check:

```bash
sudo systemctl status postgresql
```

Smoke test:

```bash
sudo -u postgres psql -c "SELECT 1;"
```

The PostgreSQL installation was verified successfully.

## 4. Networking

Checking available Docker networks:

```bash
sudo docker network ls
```

Inspecting the default `bridge` network:

```bash
sudo docker inspect bridge
```

Checking Docker network interfaces on the host:

```bash
ip a show docker0
```

Running two test containers in the default `bridge` network:

```bash
sudo docker run -dit --name test1 alpine
sudo docker run -dit --name test2 alpine
sudo docker ps
```

Checking container IP addresses:

```bash
sudo docker exec -it test1 /bin/sh
ip a
exit
sudo docker exec -it test2 /bin/sh
ip a
exit
```

Testing connectivity between containers:

```bash
sudo docker exec -it test1 /bin/sh
ping -c 3 172.17.0.3
exit
sudo docker exec -it test2 /bin/sh
ping -c 3 172.17.0.2
exit
```

Testing name resolution in the default `bridge` network:

```bash
sudo docker exec -it test1 /bin/sh
ping -c 3 test2
exit
```

Container name resolution is not available between containers on the default bridge network, so the containers can communicate by IP address but not by container name.

Creating a user-defined bridge network:

```bash
sudo docker network create testbridge
sudo docker network ls
```

Running two test containers in the user-defined network:

```bash
sudo docker rm -f test1
sudo docker rm -f test2
sudo docker run -dit --name test1 --network testbridge alpine
sudo docker run -dit --name test2 --network testbridge alpine
```

Testing connectivity by container name:

```bash
sudo docker exec -it test1 /bin/sh
ping -c 3 test2
exit
sudo docker exec -it test2 /bin/sh
ping -c 3 test1
exit
```

Removing test containers and the test network:

```bash
sudo docker rm -f test1
sudo docker rm -f test2
sudo docker network rm testbridge
sudo docker ps -a
sudo docker network ls
```

## 5. Persistent storage

To be completed.

## 6. Testing

To be completed.

## Troubleshooting

### SSH still listens on port 22 after changing `sshd_config`

After changing the SSH port in `/etc/ssh/sshd_config` to `2007` and restarting the SSH service, SSH was still listening on port 22.

The effective SSH configuration was checked with:

```bash
sudo sshd -T | grep '^port'
```

This confirmed that `sshd` was configured to use port `2007`.

The actual listening sockets were then checked with:

```bash
sudo ss -tlnp | grep ssh
```

The socket was still listening on port 22 and was managed by `systemd` through `ssh.socket`.

To apply the new port, the systemd configuration was reloaded and the SSH socket was restarted:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

The listening port was verified again with:

```bash
sudo ss -tlnp | grep ssh
```

**Key point:** `sshd -T` shows the effective SSH daemon configuration, while `ss` shows which ports are actually listening. When SSH uses systemd socket activation, changing `sshd_config` may also require restarting `ssh.socket`.

### `sudo apt update` error

After adding the `Docker` repository an error occurred:

```bash
sudo apt update
E: Malformed entry 1 in sources file /etc/apt/sources.list.d/docker.sources (URI)
E: The list of sources could not be read.
```

`docker.sources` was checked:

```bash
cat /etc/apt/sources.list.d/docker.sources
```

```

Types: deb

URIs: https://download.docker.com/linux/ubuntu

Suites: noble

Components: stable

Architectures: arm64

Signed-By: /etc/apt/keyrings/docker.asc

```

The file contained an empty line after each entry, causing `apt` to parse them as separate entries.

```bash
sudo nano /etc/apt/sources.list.d/docker.sources
```

After removing the empty lines, the file looked like this:

```bash
ruslan@docker-lab:~$ cat /etc/apt/sources.list.d/docker.sources
```

```
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: noble
Components: stable
Architectures: arm64
Signed-By: /etc/apt/keyrings/docker.asc
```

After the fix, apt update completed successfully.
