# openwrt_home
Installation Openwrt to old PC which is the Ubuntu 20.04

## 1 install Ubuntu on the laptop
### 1.1 Tools requirement:

OS image: ubuntu 20.04

Falsh OS software: balenaetcher (https://www.balena.io/etcher/) 

### 1.2 Set root passwd
```
freedom@freedom-ubuntu:~$ sudo -i
[sudo] password for freedom:

root@freedom-ubuntu:~# passwd
New password:
```

### 1.3 Change the source to tsinghua

https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/

### 1.4 Enable ssh and Allow root login

```
# update source and install ssh service
sudo apt update
sudo apt install openssh-server
sudo apt install openssh-client

# check ssh service status
sudo systemctl status ssh

# enable root to ssh
vi /etc/ssh/sshd_config
PermitRootLogin yes
```


## 2 Docker Installation

https://www.runoob.com/docker/ubuntu-docker-install.html


## 3 Create Openwrt Container 

Refer: https://github.com/crazygit/openwrt-x86-64

### 3.1 Checking the eth, which is enp7s0
```
# ifconfig
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:baff:fe95:3721  prefixlen 64  scopeid 0x20<link>
        ether 02:42:ba:95:37:21  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3  bytes 306 (306.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp7s0: flags=4419<UP,BROADCAST,RUNNING,PROMISC,MULTICAST>  mtu 1500
        inet 192.168.31.135  netmask 255.255.255.0  broadcast 192.168.31.255
        inet6 fe80::4158:fd8c:b296:b981  prefixlen 64  scopeid 0x20<link>
        ether 1c:75:08:5f:9c:c5  txqueuelen 1000  (Ethernet)
        RX packets 1875708  bytes 1456282761 (1.4 GB)
        RX errors 0  dropped 5218  overruns 0  frame 0
        TX packets 1524130  bytes 574085265 (574.0 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 3287  bytes 281386 (281.3 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3287  bytes 281386 (281.3 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

### 3.2 Enable hybrid mode of host network card (optional)
`sudo ip link set enp7s0 promisc on`

### 3.3 Create a macvlan mode network for docker. 
```
$ docker network create -d macvlan --subnet=192.168.31.0/24 --gateway=192.168.31.1 -o parent=enp7s0 openwrt-LAN

$ docker network ls |grep openwrt-LAN
21dcddacc389        openwrt-LAN         macvlan             local
```

### 3.4 Run container
Here using the esiropenwrt which includes openclash, avoid the complex approach of openclash installation.

```
# --network using the created on above step
$ docker run --restart always --name openwrt -d --network openwrt-LAN --privileged esirpg/buddha /sbin/sh

$ docker ps -a
CONTAINER ID   IMAGE           COMMAND        CREATED        STATUS        PORTS     NAMES
a7b42482a5ca   esirpg/buddha   "/sbin/init"   14 hours ago   Up 14 hours             openwrt
```

### 3.5 Configure network in Openwrt container
```
#enter openwrt container
docker exec -it openwrt /bin/sh

# config LAN interface 
$ vi /etc/config/network
config interface 'lan'
        option type 'bridge'
        option ifname 'eth0'
        option proto 'static'
        option ipaddr '192.168.31.250'
        option netmask '255.255.255.0'
        option gateway '192.168.31.1'
        option dns '192.168.31.1'
        option broadcast '192.168.31.255'

# disable DHCP for LAN
$ vi  /etc/config/dhcp
config dhcp 'lan'
        option interface 'lan'
        option ra_slaac '1'
        list ra_flags 'managed-config'
        list ra_flags 'other-config'
        option ignore '1'

$ /etc/init.d/network restart
```

**Note**
modify /etc/resolv.conf for can execute 'opkg update' 
```
vi /etc/config/network
#nameserver 127.0.0.11
#options edns0 trust-ad ndots:0
search lan
nameserver 192.168.31.1
nameserver 8.8.8.8
options edns0 trust-ad ndots:0
```


## Openwrt Services Installation

### AliyunWebDAV
Refer: https://github.com/messense/aliyundrive-webdav

Using this version for my system: aliyundrive-webdav_1.10.1-1_x86_64.ipk 

```
wget https://github.com/messense/aliyundrive-webdav/releases/download/v1.10.1/aliyundrive-webdav_1.10.1-1_x86_64.ipk
wget https://github.com/messense/aliyundrive-webdav/releases/download/v1.10.1/luci-app-aliyundrive-webdav_1.10.1_all.ipk
wget https://github.com/messense/aliyundrive-webdav/releases/download/v1.10.1/luci-i18n-aliyundrive-webdav-zh-cn_1.10.1-1_all.ipk
opkg install aliyundrive-webdav_1.10.1-1_x86_64.ipk
opkg install luci-app-aliyundrive-webdav_1.10.1_all.ipk
opkg install luci-i18n-aliyundrive-webdav-zh-cn_1.10.1-1_all.ipk
```


### Docker qbittorrent Installation
```
docker pull linuxserver/qbittorrent

docker run -d \
  --name=qbittorrent  \
  --network openwrt-LAN \
  -e WEBUI_PORT=8080 \
  -p 9821:6881 \
  -p 9821:6881/udp \
  -p 8080:8080 \
  -v /home/freedom/Downloads:/downloads \
  --restart unless-stopped \
  linuxserver/qbittorrent
  ```
  
### Docker Clash Installation
  
Refer: https://zhuanlan.zhihu.com/p/423684520

