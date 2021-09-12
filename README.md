# Rancher-Kubernetes-Cluster
These are the steps I followed to setup rancher on a VM

## Download and Install VirtualBox
Google it

## CentOS Download
https://www.osboxes.org/centos/
Download server version
login: osboxes
pwd: osboxes.org

## Configure VirtualBox Network
In VirtualBox go to File then Manage Host network
Add a new one
Configure manually - adapter
Enable server - dhcp server
Note the lower and upper bounds, these can be used as static ips in VMs
Save and exit

Example
Server Addr: 192.168.56.100
Server Mask: 255.255.255.0
Lower: 192.168.56.101
Upper: 192.168.56.254

## Create VM and Configure Network
Create a new VM in VirtualBox using the CentOS vdi
For master node: Increase RAM to at least 4000
For master node: Increase CPUs to 2
For all:
Under network tab should have default NAT network
Add additional network adapter
Host-only Adapter
Select network that was created in VirtualBox Host network manager
If youre using multiple nodes make sure VMs have different MAC Addresses

## Launch VM and login, configure NIC
https://mikesmithers.wordpress.com/2018/11/17/virtualbox-configuring-a-host-only-network/
```bash
sudo nano /etc/sysconfig/network-scripts/ifcfg-enp0s8

TYPE=ETHERNET
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=yes
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
DEVICE=enp0s8
ONBOOT=yes
IPADDR=192.168.56.101
PREFIX=24
GATEWAY=192.168.56.1
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_PRIVACY=no

cat /etc/sysconfig/network-scripts/ifcfg-enp0s8
````

UPDATE THE HOSTNAME TO WHATEVER YOU WANT
```bash
sudo nano /etc/hostname
magneto

cat /etc/hostname

sudo reboot
```

If setting up multiple nodes, also add each other to the hosts file so you can ping by name
```bash
sudo nano /etc/hosts
ADD
192.168.56.101 magneto
192.168.56.102 wolverine
```

## Configure hosts file on host machine
This will allow you to ping using the hostname instead of ip
```bash
sudo nano /etc/hosts
ADD
192.168.56.101 master
192.168.56.102 node1
```
I should now be able to ping from host machine to VMs 
Should be able to ssh
should be able to ping from VM to VM

## Enable sudo with no pwd
Easier but less secure!
```bash
$ su -
# echo -e “osboxes\tALL=(ALL)\tNOPASSWD: ALL" > /etc/sudoers.d/020_sudo_for_me

# cat /etc/suders.d/020_sudo_for_me
osboxes ALL=(ALL) NOPASSWD: ALL
# exit
```

## Install docker
https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-centos-7
https://docs.docker.com/engine/install/linux-postinstall/
REMOVE ANY PREEXISTING DOCKER
```bash
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
sudo yum check-update
```
NOT SURE WHY BUT I HAD TO RUN NEXT LINE FOR IT WORK
```bash
sudo yum erase podman buildah
```
ACTUAL INSTALL
```bash
sudo curl -fsSL https://get.docker.com/ | sh
sudo systemctl start docker
sudo docker run hello-world
```
DISABLE SUDO FOR DOCKER
```bash
sudo usermod -aG docker $(whoami)
```
START DOCKER ON BOOT
```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

## Install rancher
https://rancher.com/quick-start/
https://rancher.com/docs/rancher/v1.6/en/quick-start-guide/
I installed the following version specifically (not as resource intensive)
Only have to install rancher on main node/master, workers do not need it
```bash
docker run -d --restart=unless-stopped -p 80:80 -p 443:443 -v /opt/rancher:/var/lib/rancher  --name rancher rancher/rancher:v2.4.0-alpha1
```
should be able to open browser on host machine and pull up rancher
note the IP addrress and pwd

## Use rancher UI to create cluster
Use Calico provider

Master node should be like this (magneto)
```bash
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.4.0-alpha1 --server https://192.168.56.101 --token f9wz7sjrjkjndq2vk2dxz5d4d5xsjcbswnklm7l2lb4ljhlt59x9s2 --ca-checksum d439de631e39fa6994c7fd7600463f55e19493bb053389e4dac639dff25ba097 --etcd --controlplane --worker
```

worker nodes should look like this
will most likely have to specify the IP by adding the following line
```bash
--address 192.168.56.102 --internal-address 192.168.56.102
```
```bash
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.4.0-alpha1 --server https://192.168.56.101 --address 192.168.56.102 --internal-address 192.168.56.102 --token f9wz7sjrjkjndq2vk2dxz5d4d5xsjcbswnklm7l2lb4ljhlt59x9s2 --ca-checksum d439de631e39fa6994c7fd7600463f55e19493bb053389e4dac639dff25ba097 --worker
```

Success
2 new nodes have registered 

If master (magneto) is having issues launching etcd/control pane because the connection is refused, may need to configure firewall
Firewall on Magneto
https://stackoverflow.com/questions/58268197/kubernetes-dial-tcp-myip10250-connect-no-route-to-host
```bash
sudo firewall-cmd --add-port=6443/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all  # you should see that port `10250` is updated
```

## Commands/Notes
vi
```bash
sudo visudo
[esc] - to return
i - to edit
:wq - to save
:q! - to not save
```
System
```bash
sudo shutdown now
sudo reboot
```
Docker logs
```bash
sudo docker logs -f <CONTAINER_ID>
```
Docker
```bash
sudo systemctl start docker
sudo docker ps
sudo docker container stop

STOP ALL
docker stop $(docker ps -q)

REMOVE ALL
docker system prune -a

DISABLE SUDO
sudo usermod -aG docker $(whoami)

docker start rancher
```
