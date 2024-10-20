# Cluster-in-VirtualBox

## Goals
In this tutorial, we are going to create a set of four Linux virtual machines.

- Each machine is capable of connecting to the internet and able to connect with each other privately as well as can be reached from the host machine.
- Our machines will be named master, node01, node02,..., node0X.
- The first machine will act as a master node and will have 1vCPUs, 2GB of RAM and 25 GB hard disk.
- The other machines will act as workers and they will have 1vCPUs, 1GB of RAM and 10 GB hard disk.
- We will assign our machines static IP address in the internal network: 192.168.0.1, 192.168.0.22, 192.168.0.23, .... 192.168.0.XX.

<p align="center">
  <img src="https://github.com/user-attachments/assets/9bddf35b-147c-49d5-9778-f1a68295bcca" width="500">
</p>

## Prerequisite Setup
- VirtualBox must be installed on your system (Windows/Linux/Mac).
- Download the Ubuntu 24.10 image from Ubuntu's website (https://ubuntu.com/download/server).


## Template VM Creation
**Objective**: Create a template VM that can be cloned to deploy the cluster.

### Creation
- In VirtualBox click _**Machine>New**_.
- Configure name and operating system of the machine (you must select the Ubuntu image previously downloaded).
<p align="center">
  <img src="https://github.com/user-attachments/assets/406e5466-695b-4b8a-a7ef-5107f98867e9" width="700">
</p>

- Configure username and password.
<p align="center">
  <img src="https://github.com/user-attachments/assets/4578060f-7f05-44e9-b80a-3d4831a121e3" width="700">
</p>

- Configure hardware (lets select 1000 MB of RAM memory and 1 CPU).
<p align="center">
  <img src="https://github.com/user-attachments/assets/d87a6cf4-a861-45d0-8dbb-b4b1e357c675"  width="700">

- Configure hard disk (select 25 GB if it is not already the default option in your machine).
<p align="center">
  <img src="https://github.com/user-attachments/assets/6345406a-ef18-4c57-acda-7dcba1c6d5b5"  width="700">
</p>

After clicking _Finish_, the machine will start. Lets wait ultil it is fully built so we can test it.


### Package installation
Run these commands to update apt package manager:
```
sudo apt update
sudo apt upgrade
```

Then, install some aditional packages running
```
sudo apt install net-tools gcc make
```

After setting up the template VM, shut it down:
```
sudo shutdown -h now
```

The template machine should be ready now for cloning.


## Master/Nodes creation & Network adapters configuration
### Machine clonation
First, lets clone the template machine so we can work in the real nodes of our cluster.
- Right click on _template_ machine in VirtualBox and then select _Clone_.
- Set a name for you clone.
<p align="center">
  <img src="https://github.com/user-attachments/assets/19d28453-aaf8-45d6-95bb-b722caa1bb33"  width="700">
</p>

- Repeat any number of times to create the worker nodes (for now, I will only create one worker named _node01_).


### Network adapters configuration
Once it is created, lets configure the two network adapters:
1. Adapter 1 (NAT): This connects the VM to the host’s network and allows internet access.
2. Adapter 2 (Internal network): This is used for communication between the VMs on a private internal network, where each VM is assigned a static IP address.
   
- Right click in the virtual machine (_master_/_node01_) and then **_Settings>Network_**. Check that the first adapter is attached to NAT.
<p align="center">
  <img src="https://github.com/user-attachments/assets/d7b8041d-f912-4587-87c1-5e6b8789d9b3"  width="700">
</p>

- Select _Adapter 2_, click on _Enable Network Adapter_ and attach it to _Internal Network_. Then select a name for the network that must be the same for all the nodes (e.g., _clustervimnet_).
<p align="center">
  <img src="https://github.com/user-attachments/assets/6fff35b0-a7af-4db3-b422-3b38bacff0b6"  width="700">
</p>

- Save the settings and repeat the configuration for the other nodes.



## Network Configuration for Master Node
The master node will act as the control point for the cluster, managing DNS (domain name system) and DHCP (dynamic host configuration protocol) services.

### Configure the network file
You need to configure the second network adapter (Adapter 2) to assign a static IP address. This IP address will allow the master node to communicate with the other worker nodes on the internal network.

After starting and logging into the master VM, find the network interfaces available on your VM using
```
ip link show
```

The output will be something similar to this:
<p align="center">
  <img src="https://github.com/user-attachments/assets/79386b9a-951c-4d61-99af-00e8fa3a56c0"  width="700">
</p>

where

- **enp0s3** is the first adapter (NAT).
- **enp0s8** is the second adapter (internal network).

We want to configure enp0s8 to have a static IP, like 192.168.0.1. To do this, lets edit the Netplan configuration file.
- Open the file with:
```
sudo vim /etc/netplan/50-cloud-init.yaml
```
- Add the necessary lines to make it look like this:
```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
     dhcp4: no
     addresses: [192.168.0.1/24]
  version: 2
```

Finally, apply the changes with
```
sudo netplan apply
```


### Configure hosts
To make it easier to identify machines in the cluster, lets change the hostname of the master node. Open the host file with:
```
sudo vim /etc/hostname
```
and change its content to _master_.

We also need to define the static IP addresses and hostnames for all nodes in the cluster in the `/etc/hosts` file. This will allow the master node to resolve the IP addresses of the worker nodes by name. Open the file with:
```
sudo vim /etc/hosts
```
and edit it until looks like this:
```yaml
127.0.0.1 localhost
192.168.0.1 master

192.168.0.22 node01

# The following lines are deriable for IPv6 capable hosts
...
```
If you have more than one worker node, be sure to assign them an IP (192.168.0.23 node02, ...).



##  Setting Up Port Forwarding for SSH

### Port forwarding rule
To connect to the VMs (e.g., the master node) from your host machine, VirtualBox requires port forwarding rules to map ports from the guest VMs to your host machine. In this case, you’ll set up a rule to forward SSH traffic from port 2222 on the host to port 22 on the master node.

- Open VirtualBox and right click on the _master_ machine, then **_Settings>Network>Port Forwarding>Add_**.
- Fill the gaps following the table.

| Namer  | Protocol | Host IP | Host Port | Guest IP | Gest Port |
| ------------- | ------------- | ------------- | ------------- | ------------- | ------------- |
| ssh  | TCP  | 127.0.0.1 | 2222 | | 22 |

- Apply the changes and restart the VM.


### SSH service
In order to make SSH work, we need first to install its package.
```
sudo apt install openssh-server
```

Lets check it is enabled now by running the command:
```
sudo systemctl status ssh
```

To ensure it starts on boot:
```
sudo systemctl enable ssh
```


### Passwordless SSH
To avoid entering a password each time you connect via SSH, you can configure SSH key-based authentication.

First, on your host machine, generate an SSH key pair (if you don’t have one). I will be working in Windows PowerShell.
```
ssh-keygen
```

Now, copy the generated public key (mine is _id_ed25519.pub_ but yours can be different) from your host to the master node (your VM must be running).
```
cd .ssh
scp -P 2222 id_ed25519.pub admin@127.0.0.1:~
```

If your host machine shows a HUGE warning telling you that you can be suffering a man-in-the-middle attack, don't worry, you are that man. When you connect to an SSH server for the first time, the server’s host key is stored in the known_hosts file on your client machine. If the host key of the server changes (which can happen, for example, if you rebuild the VM), SSH assumes this could be a security issue. You need to remove the outdated or incorrect host key from your known_hosts file and run the _scp_ command again.
```
ssh-keygen -R [127.0.0.1]:2222
```

At this point, you should have been able to connect you VM by SSH. Now that the public key has been copied into your home directory, lets add it to the `authorized_keys` file.
```
cat id_ed25519.pub >> .ssh/authorized_keys
rm id_ed25519.pub
```

To verify everything works fine, just open a new terminal in your host machine and lunch an SSH connection
```
ssh -p 2222 admin@127.0.0.1
```

If everything is set up correctly, you will be logged in without needing to enter a password.





## DHCP and DNS
The goal of this section is to set up the master node to dynamically assign IP addresses and manage DNS resolution for the worker nodes.

First, install `dnsmasq`, which is a lightweight DHCP and DNS server. It will serve the internal network, assigning dynamic IPs to the nodes and resolving hostnames.
```
sudo apt install dnsmasq -y
```

Then, we need to configure the `/etc/dnsmasq.conf` file. The configuration given in the tutorial is minimal, but it’s sufficient for basic DHCP and DNS operations.
```
sudo vim /etc/dnsmasq.conf
```

Here are the key settings:

- **port=53**: This configures dnsmasq to listen on port 53, the standard port for DNS.
- **interface=enp0s8**: This defines the network interface on which `dnsmasq` listens. Ensure that the internal network interface (in my case, enp0s8) is specified.
- **listen-address**: Specifies the IP addresses `dnsmasq` will listen on. These include loopback (127.0.0.1) and the IP of the master node on the internal network (192.168.0.1).
- **dhcp-range**: This defines the IP range that dnsmasq will assign to the worker nodes, with a lease time of 12 hours.
- **dhcp-option=option:dns-server,192.168.0.1**: Specifies the DNS server for the DHCP clients, which is the master node itself.

```yaml
port=53
bogus-priv
strict-order
interface=enp0s8
listen-address=::1,127.0.0.1,192.168.0.1
bind-interfaces
log-queries
log-dhcp
dhcp-range=192.168.0.22,192.168.0.28,255.255.255.0,12h
dhcp-option=option:dns-server,192.168.0.1
dhcp-option=3

# Read entries from /etc/hosts and serve them as part of the DNS
addn-hosts=/etc/hosts
```

To ensure proper DNS resolution, create a symbolic link to the system `resolv.conf` file, which uses `systemd-resolved`.
```
sudo ln -fs /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

We now need to ensure that the master node has the correct network configuration for both interfaces. Edit the network file to make it similar to this:
```
sudo vim /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses: [192.168.0.1/24]
      nameservers:
        addresses: [192.168.0.1]
  version: 2
```
```
sudo netplan apply
```

If we want to control how `dnsmasq` interacts with `resolvconf`, we need to uncommend a couple of lines in the `/etc/default/dnsmasq` file. Lets open the file:
```
sudo vim /etc/default/dnsmasq
```

And uncomment the following lines:
```yaml
IGNORE_RESOLVCONF=yes
DNSMASQ_EXCEPT="lo"
```

Apply the changes by restarting the services.
```
sudo systemctl restart dnsmasq systemd-resolved
```

After restarting, check that dnsmasq is running without issues. The service should be enabled and active.
```
sudo systemctl status dnsmasq
```

To check if the DNS server is resolving names correctly, use the `host` command:
```
host node01
```

The ouptut should be similar to this:
<p align="center">
  <img src="https://github.com/user-attachments/assets/84466e1f-10fd-49c3-b21c-4efa6d63f7b4"  width="400">
</p>

If your machine does not find the node:
<p align="center">
  <img src="https://github.com/user-attachments/assets/bbd58bcb-c25c-45e6-a9e6-38c2026e7500"  width="400">
</p>

you should try to either restart the `dnsmasq` service or reboot your VM.
```
sudo systemctl restart dnsmasq
```

NOTE: Afther bootstrap may happen the `dnsmasq` service start before the interfaces, just restart the service.
```
sudo systemctl restart dnsmasq systemd-resolved
```



## Port Forwarding and NAT in the Internal Network
In order to connect to the internet from the worker nodes using the master node as a gateway, we need to configure port forwarding in our master node.

Lets create a new file and write the following line:
```
sudo vim /etc/sysctl.d/99-ipforward.conf
```
```yaml
net.ipv4.ip_forward=1
```

Apply changes inmmediately
```
sudo sysctl --system
```

Verify that IP forwarding is enabled by running the following command. If the ouput is "1", everything is fine by now.
```
cat /proc/sys/net/ipv4/ip_forward
```

We need now to ensure that the IP tables are configured to allow NAT (Network Address Translation) between network interfaces.

`iptables-persistent` is a package that automatically saves your current iptables rules and loads them when the system boots. Lets install it (press _yes_ all the times it is needed):
```
sudo apt install iptables-persistent
```

Now it's time to set up a NAT rule for the outgoing interface (enp0s3) and save it:
```
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo netfilter-persistent save
```



## Distributed File System
To build a cluster, you need a shared filesystem that all nodes can access.

Lets begin installing the NFS kernel server package.
```
sudo apt install nfs-kernel-server
```

Then create a shared directory that all nodes will access.
```
sudo mkdir /shared
sudo chmod 777 /shared
```

Open the NFS exports file and specify the directory and network permissions.
- **rw**: Grants read and write permissions.
- **sync**: Ensures changes are written to disk immediately.
- **no_root_squash**: Allows root users on clients to access files as root.
- **no_subtree_check**: Disables subtree checking for better performance.
```
sudo vim /etc/exports
```
```yaml
/shared/ 192.168.0.0/255.255.255.0(rw,sync,no_root_squash,no_subtree_check)
```
This allows all machines on the internal network (192.168.0.0/24) to access the shared directory with read/write permissions.

Enable and start the NFS server.
```
sudo systemctl enable nfs-kernel-server
sudo systemctl restart nfs-kernel-server
```

We can now create a test file in the new folder
```
touch /shared/ciao_mondo.txt
```


## Worker Nodes Configuration

### Network configuration
First we need to modify the network file.
```
sudo vim /etc/netplan/50-cloud-init.yaml
```
```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: true
      dhcp4-overrides:
        use-dns: no
    enp0s8:
      dhcp4: true
      dhcp-identifier: mac
      nameservers:
        addresses: [192.168.0.1]
      routes:
        - to: 0.0.0.0/0
          via: 192.168.0.1
```

For the first adapter (enp0s3):
- **dhcp4: true**: This allows the node to obtain an IP address dynamically via DHCP.
- **dhcp4-overrides.use-dns**: no: This ensures that the DNS from this interface is not used, as you want to use the master node's DNS for internal cluster communication.
For the second adapter (enp0s8):
- **dhcp4: true**: The interface will use DHCP to get an IP address. This IP will be in the range defined by the master node's DHCP server (set up using `dnsmasq`).
-** dhcp-identifier: mac**: This ensures that the DHCP server assigns an IP address based on the MAC address, allowing for stable IP assignments.
- **nameservers.addresses: [192.168.0.1]**: The DNS server for the internal network will be the master node at 192.168.0.1. This ensures that the DNS queries for the cluster are handled by the master node.

Apply the configuration:
```
sudo netplan apply
```

Ensure the /etc/hostname file is cleared (you must empty it).
```
sudo vim /etc/hostname
```

Finally, set the DNS server.
```
sudo ln -fs /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

Now it is time to reboot our VM. In order to verify that the nodes gets an IP from the DHCP, lets run the command:
```
hostname -I
```
You should see two IP addresses:
- One from the NAT network (e.g., 10.0.2.15).
- One from the internal network (in my case in the range [192.168.0.22. 192.168.0.28]).


By this point you should be able to correctly connect your master node from the worker node. You can try it by
```
ping 192.168.0.1
```
Moreover, you should also be able to connect to the internet ONLY using the second network adapter (enp0s8). You can manually deactivate the NAT adapter (enp0s3) from VirtualBox (the machine must be off) and try to connect _google.com_:
```
ping google.com
```

If it works, that means that:
- You are getting an IP dinamically (so the DHCP server is working properly).
- The domain is being resolved (so the DNS server is working properly).
- Your master node is able to return the packages from the destination to you (NAT and Port Forwarding are working properly).


### File System configuration (mounting point)
Lets install the NFS client on each worker node.
```
sudo apt install nfs-common
```

Create a mount point (where the shared folder from the master will be mounted).
```
sudo mkdir /shared
```

Mount the shared directory from the master node to this folder.
```
sudo mount 192.168.0.1:/shared /shared
```

To check if everything went fine, just show the content of the folder. The file `ciao_mondo.txt` should appear.
```
ls /shared
```

NOTE: This type of mounting only works while the machine is running. After rebooting the mount point will disappear.



### File System configuration (automatic mount)
Install AutoFS to automatically mount the shared directory on boot, install the AutoFS package.
```
sudo apt -y install autofs
```

Edit the `auto.master` configuration file to include the mount points adding the following line:
```
sudo vim /etc/auto.master
```
```
/- /etc/auto.mount
```

Create the AutoFS configuration file (`auto.mount`) to define the NFS mount. Add the following configuration:
```
sudo vim /etc/auto.mount
```
```yaml
/shared -fstype=nfs,rw  192.168.0.1:/shared
```

Finally, restart AutoFS service to apply changes.
```
sudo systemctl restart autofs
```

Now the `shared` folder shoud remain mounted even after rebooting your node.

