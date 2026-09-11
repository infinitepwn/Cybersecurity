# 配置环境

## 创建OpenWRT 虚拟机

``` bash
VBoxManage convertfromraw openwrt-armsr-armv8-generic-ext4-combined-efi.img outputfile.vdi --format VDI
```

![image-20260906182902557](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260906182902557.png)

新建虚拟机

新版本virtualbox已经和教程里不太一样

没有linux选项，就选了个other Linux

<img src="https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260901081736897.png" alt="image-20260901081736897" style="zoom:50%;" />

使用已有的文件

<img src="https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260901082731497.png" alt="image-20260901082731497" style="zoom:33%;" />

启动

<img src="https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260906184014710.png" alt="image-20260906184014710" style="zoom:33%;" />

arm的桌面ubuntu镜像国内镜像站都找不到

谷歌搜寻发现是有的

<img src="https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260906183714171.png" alt="image-20260906183714171" style="zoom:33%;" />

安装ubuntu

<img src="https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260907113200703.png" alt="image-20260907113200703" style="zoom:25%;" />

需要给4GB以上内存，不然copying files那里会卡死



配置网络

给attacker

<img src="https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260907131148440.png" alt="image-20260907131148440" style="zoom: 25%;" />

victim同理

<img src="https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260907131221994.png" alt="image-20260907131221994" style="zoom:25%;" />

Server

<img src="https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260907131257746.png" alt="image-20260907131257746" style="zoom: 33%;" />

但是由于这个virtualbox占用的存储比较大，最后还是改用docker来做

<img src="https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908083339826.png" alt="image-20260908083339826" style="zoom:25%;" />

在安装完虚拟机之后，存储达到了244GB，实在是不够用，只能全删了

## 在Docker中部署

编辑docker-compose.yml

### openwrt部署

```yaml
services:
  router:
    build:
      context: ./openwrt
    container_name: openwrt
    hostname: router
    privileged: true
    restart: unless-stopped

    networks:
      wan:
        interface_name: eth0
        ipv4_address: 172.19.0.2
        gw_priority: 100

      intnet1:
        interface_name: eth1
        ipv4_address: 192.168.10.1

      intnet0:
        interface_name: eth2
        ipv4_address: 192.168.26.1

```

### 配置ubuntu

```yaml

  victim:
    image: ubuntu:20.04
    platform: linux/arm64
    container_name: victim
    hostname: victim
    cap_add:
      - NET_ADMIN
      - NET_RAW

    networks:
      intnet0:
        interface_name: eth0
        ipv4_address: 192.168.26.3

    dns:
      - 8.8.8.8
      - 8.8.4.4

    command: >
      bash -c "
      apt-get update &&
      DEBIAN_FRONTEND=noninteractive apt-get install -y
      iproute2
      iputils-ping
      net-tools
      curl
      tcpdump
      vim &&
      ip route del default &&
      ip route add default via 192.168.26.1 &&
      tail -f /dev/null
      "

  attacker:
    image: ubuntu:20.04
    platform: linux/arm64
    container_name: attacker
    hostname: attacker
    cap_add:
      - NET_ADMIN
      - NET_RAW

    networks:
      intnet0:
        interface_name: eth0
        ipv4_address: 192.168.26.2

    dns:
      - 8.8.8.8
      - 8.8.4.4

    command: >
      bash -c "
      apt-get update &&
      DEBIAN_FRONTEND=noninteractive apt-get install -y
      iproute2
      iputils-ping
      net-tools
      curl
      tcpdump
      nmap
      netcat-openbsd
      vim &&
      ip route del default &&
      ip route add default via 192.168.26.1 &&
      tail -f /dev/null
      "

  server:
    image: ubuntu:20.04
    platform: linux/arm64
    container_name: server
    hostname: server
    cap_add:
      - NET_ADMIN
      - NET_RAW

    networks:
      intnet1:
        interface_name: eth0
        ipv4_address: 192.168.10.2

    dns:
      - 8.8.8.8
      - 8.8.4.4

    command: >
      bash -c "
      apt-get update &&
      DEBIAN_FRONTEND=noninteractive apt-get install -y
      iproute2
      iputils-ping
      net-tools
      curl
      tcpdump
      vim &&
      ip route del default &&
      ip route add default via 192.168.10.1 &&
      tail -f /dev/null
      "

networks:
  wan:
    driver: bridge
    ipam:
      config:
        - subnet: 172.19.0.0/16
          gateway: 172.19.0.1

  intnet1:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.10.0/24
          gateway: 192.168.10.254

  intnet0:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.26.0/24
          gateway: 192.168.26.254

```

然后部署

```bash
docker compose up -d
```

如果想要进入某个机器，就

```bash
docker exec -it server bash
```
我们利用ping来测试一下连通性

![image-20260908081541943](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908081541943.png)
ping了三次，首先第一次是在把防火墙stop的情况下ping的，可以ping通，第二次是restart了防火墙，显示不可达Destination Port Unreachable，第三次是修改了防火墙之后，又能ping通了

![image-20260908082240360](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908082240360.png)
修改防火墙如下

![image-20260908082335633](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908082335633.png)

注：此处与指导文件有出入，新的openwrt固件对格式要求变更了，需要用list network 'lan1'

### victim到attacker的跳数

![image-20260908082948472](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908082948472.png)

### victim到server的跳数

![image-20260908083045058](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908083045058.png)

### 给attacker和server安装工具

```bash
apt install tcpdump wireshark python3-scapy
```


![image-20260908081016956](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908081016956.png)



## 实验一

按照网络拓扑，用如下部署

![image-20260908085321564](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908085321564.png)

```yaml
services:
  route:
    build:
      context: ./openwrt
    container_name: route
    hostname: route
    privileged: true
    restart: unless-stopped

    networks:
      internal:
        interface_name: eth0
        ipv4_address: 192.168.3.1

      external:
        interface_name: eth1
        ipv4_address: 192.168.4.1
```

对于其他的机器，也是同理，command部分是因为docker起的ubuntu缺少很多工具，需要单独安装，另外配置一下路由

```yaml
  client:
    image: ubuntu:20.04
    platform: linux/arm64
    container_name: client
    hostname: client

    cap_add:
      - NET_ADMIN
      - NET_RAW

    networks:
      internal:
        interface_name: eth0
        ipv4_address: 192.168.3.10

    command: >
      bash -c "
      apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y iproute2 iputils-ping tcpdump net-tools && ip route del default && ip route add default via 192.168.3.1 && tail -f /dev/null
      "

  internal-server:
    image: ubuntu:20.04
    platform: linux/arm64
    container_name: internal-server
    hostname: internal-server

    cap_add:
      - NET_ADMIN
      - NET_RAW

    networks:
      internal:
        interface_name: eth0
        ipv4_address: 192.168.3.200

    command: >
      bash -c "
      apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y iproute2 iputils-ping tcpdump net-tools && ip route del default && ip route add default via 192.168.3.1 && tail -f /dev/null
      "

  external-server:
    image: ubuntu:20.04
    platform: linux/arm64
    container_name: external-server
    hostname: external-server

    cap_add:
      - NET_ADMIN
      - NET_RAW

    networks:
      external:
        interface_name: eth0
        ipv4_address: 192.168.4.105

    command: >
      bash -c "
      apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y iproute2 iputils-ping tcpdump net-tools && ip route del default && ip route add default via 192.168.4.1 && tail -f /dev/null
      "

```

然后配置网络

注，这里的gateway是docker虚拟网络必须占用的，和实验无关

```yaml
networks:

  internal:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 192.168.3.0/24
          gateway: 192.168.3.254

  external:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 192.168.4.0/24
          gateway: 192.168.4.254
```

<img src="https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908090300521.png" alt="image-20260908090300521" style="zoom:25%;" />

### openwrt网络设置

```bash
 docker exec -it route sh
```

进入shell之后

配置network

```
 vi /etc/config/network
```

如下

```yaml
config interface 'lan'
        option device 'eth0'  
        option proto 'static'
        list ipaddr '192.168.3.1'   
        option netmask '255.255.255.0'
        option ip6assign '60'
                      
config interface 'wan'      
        option device 'eth1'
        option proto 'static'
        option ipaddr '192.168.4.1'
        option netmask '255.255.255.0'
```

![image-20260908091116294](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908091116294.png)

修改防火墙

```yaml
config zone
        option name             wan
        list   network          'wan'
        list   network          'wan6'
        option input            ACCEPT
        option output           ACCEPT
        option forward          DROP
        option masq             1
        option mtu_fix          1
```



###  1.  子网内部局域网流量分析

```bash
docker exec -it client bash
```

启动失败了

![image-20260908091542281](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908091542281.png)

打开docker看一下什么问题

问题在于我们前面给external配置了一个虚假的外网，导致连不上网，嗯，我们得提前把工具装好，再在实验环境里做

![image-20260908091524694](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908091524694.png)

创建一个普通网络：

```
docker network create temp-wan
```

接到 client：

```
docker network connect temp-wan client
```

然后重新执行

```bash
apt update
apt install -y iproute2 iputils-ping tcpdump net-tools iputils-arping vim
```

装完后断开

```bash
docker network disconnect temp-wan client
```

设置路由

```bash
ip route add default via 192.168.3.1
```

![image-20260908093428919](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908093428919.png)

其他两个同理

![image-20260908094005432](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908094005432.png)

使用命令查看

```bash
ifconfig
ip a
tcpdump -D
arp -a
```

<img src="https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908094203387.png" alt="image-20260908094203387" style="zoom:25%;" />

<img src="https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908094241575.png" alt="image-20260908094241575" style="zoom:50%;" />

用tcpdump 监听端口eth0，观察所有流量

```bash
tcpdump -i eth0 -n -e
```

在client上打开另一个终端窗口，对Internal Server执行ping操作，执行前为避免已有ARP缓存内容带来的影响，需要先清空ARP缓存内容

```bash
ip neigh flush all
ping 192.168.3.200
```

![image-20260908094606266](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908094606266.png)

提取一下信息就是

listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

01:45:06.623949 8e:51:af:9f:83:a0 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.3.200 tell 192.168.3.10, length 28

01:45:06.624164 8e:51:af:9f:83:a0 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.3.200 tell 192.168.3.10, length 28

01:45:06.624263 a2:37:f9:19:02:a8 > 8e:51:af:9f:83:a0, ethertype ARP (0x0806), length 42: Reply 192.168.3.200 is-at a2:37:f9:19:02:a8, length 28

01:45:06.624290 8e:51:af:9f:83:a0 > a2:37:f9:19:02:a8, ethertype IPv4 (0x0800), length 98: 192.168.3.10 > 192.168.3.200: ICMP echo request, id 9, seq 1, length 64

01:45:06.624865 a2:37:f9:19:02:a8 > 8e:51:af:9f:83:a0, ethertype IPv4 (0x0800), length 98: 192.168.3.200 > 192.168.3.10: ICMP echo reply, id 9, seq 1, length 64

| 操作   | ARP请求 源MAC       | ARP请求 目标MAC     | ARP响应 源MAC       | ARP响应 目标MAC     | ICMP ECHO请求 源IP | ICMP ECHO请求 目标IP | ICMP ECHO响应 源IP | ICMP ECHO响应 目标IP |
| ------ | ------------------- | ------------------- | ------------------- | ------------------- | ------------------ | -------------------- | ------------------ | -------------------- |
| 子网内 | `8e:51:af:9f:83:a0` | `ff:ff:ff:ff:ff:ff` | `a2:37:f9:19:02:a8` | `8e:51:af:9f:83:a0` | `192.168.3.10`     | `192.168.3.200`      | `192.168.3.200`    | `192.168.3.10`       |

## 跨子网

在Client上清除ARP缓存，然后执行tcpdump（即步骤（2））

在Client上对External Server执行ping操作

```bash
ip neigh flush all
tcpdump -i eth0 -n -e
ping 192.168.4.105
```

但是发现ping不通

查找原因是因为docker的internal会形成一个隔离环境，不允许通过

![image-20260908132736093](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908132736093.png)

需要将这一行删掉，然后重新compose

![image-20260908141943134](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908141943134.png)

得到数据

| 操作   | ARP请求 源MAC       | ARP请求 目标MAC     | ARP响应 源MAC       | ARP响应 目标MAC     | ICMP ECHO请求 源IP | ICMP ECHO请求 目标IP | ICMP ECHO响应 源IP | ICMP ECHO响应 目标IP |
| ------ | ------------------- | ------------------- | ------------------- | ------------------- | ------------------ | -------------------- | ------------------ | -------------------- |
| 跨子网 | `a6:18:0c:94:bd:14` | `ff:ff:ff:ff:ff:ff` | `12:9d:05:1d:c8:16` | `a6:18:0c:94:bd:14` | `192.168.3.10`     | `192.168.4.105`      | `192.168.4.105`    | `192.168.3.10`       |

重做一下前面的实验

![image-20260908141642712](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908141642712.png)

| 操作   | ARP请求 源MAC       | ARP请求 目标MAC     | ARP响应 源MAC       | ARP响应 目标MAC     | ICMP ECHO请求 源IP | ICMP ECHO请求 目标IP | ICMP ECHO响应 源IP | ICMP ECHO响应 目标IP |
| ------ | ------------------- | ------------------- | ------------------- | ------------------- | ------------------ | -------------------- | ------------------ | -------------------- |
| 子网内 | `a6:18:0c:94:bd:14` | `ff:ff:ff:ff:ff:ff` | `86:25:b4:ee:75:2b` | `a6:18:0c:94:bd:14` | `192.168.3.10`     | `192.168.3.200`      | `192.168.3.200`    | `192.168.3.10`       |

## 掩码配置错误流量分析

首先清空 Client上的ARP缓存，若不生效可重启虚拟机：

```bash
ip neigh flush all
```

将Client的子网掩码修改为28;

```bash
ip addr del 192.168.3.10/24 dev eth0
ip addr add 192.168.3.10/28 dev eth0

ip route replace default via 192.168.3.1 dev eth0
```

![image-20260908141038442](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908141038442.png)

然后重复上面步骤

![image-20260908142218516](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260908142218516.png)

| 操作   | ARP请求 源MAC       | ARP请求 目标MAC     | ARP响应 源MAC       | ARP响应 目标MAC     | ICMP ECHO请求 源IP | ICMP ECHO请求 目标IP | ICMP ECHO响应 源IP | ICMP ECHO响应 目标IP |
| ------ | ------------------- | ------------------- | ------------------- | ------------------- | ------------------ | -------------------- | ------------------ | -------------------- |
| 掩码错 | `a6:18:0c:94:bd:14` | `ff:ff:ff:ff:ff:ff` | `12:9d:05:1d:c8:16` | `a6:18:0c:94:bd:14` | `192.168.3.10`     | `192.168.3.200`      | `192.168.3.200`    | `192.168.3.10`       |

主要区别就是响应的源mac地址和跨子网的一样了，都发给网关（也就是route）

## 实验二

### 搭建拓扑环境

openwrt和上面一致，多了一个dns的端口

```yaml
services:
  route:
    build:
      context: ./openwrt
    container_name: route
    hostname: route
    privileged: true
    restart: unless-stopped

    networks:
      internal:
        interface_name: eth0
        ipv4_address: 192.168.3.1

      external:
        interface_name: eth1
        ipv4_address: 192.168.4.1

      dns_net:
        interface_name: eth2
        ipv4_address: 192.168.2.1
```

配置network

```bash
config interface 'lan'
    option device 'eth0'
    option proto 'static'
    option ipaddr '192.168.3.1'
    option netmask '255.255.255.0'

config interface 'wan'
    option device 'eth1'
    option proto 'static'
    option ipaddr '192.168.4.1'
    option netmask '255.255.255.0'

config interface 'dnsnet'
    option device 'eth2'
    option proto 'static'
    option ipaddr '192.168.2.1'
    option netmask '255.255.255.0'
/etc/init.d/network restart
```

防火墙

```bash
config interface 'dnsnet'
        option device 'eth2'
        option proto 'static'
        option ipaddr '192.168.2.1'
        option netmask '255.255.255.0'
```

client

```yaml
client:
  image: kasmweb/ubuntu-focal-desktop:1.16.0
  container_name: client
  hostname: client

  privileged: true

  ports:
    - "6901:6901"

  environment:
    VNC_PW: password

  networks:
    internal:
      interface_name: eth0
      ipv4_address: 192.168.3.10
```

dns

```yaml
  dns:
    build:
      context: ./dns
    container_name: dns
    hostname: dns

    cap_add:
      - NET_ADMIN
      - NET_RAW

    depends_on:
      - route

    networks:
      dns_net:
        interface_name: eth0
        ipv4_address: 192.168.2.53

    command:
      - /bin/bash
      - -c
      - |
        ip route del default || true
        ip route add default via 192.168.2.1 dev eth0
        exec dnsmasq --no-daemon
```

server

```yaml
  server:
    build:
      context: ./server
    container_name: server
    hostname: server

    cap_add:
      - NET_ADMIN
      - NET_RAW

    depends_on:
      - route

    networks:
      external:
        interface_name: eth0
        ipv4_address: 192.168.4.105

    command:
      - /bin/bash
      - -c
      - |
        ip route del default || true
        ip route add default via 192.168.4.1 dev eth0
        exec nginx -g 'daemon off;'
```

network

```yaml
networks:
  internal:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.3.0/24
          gateway: 192.168.3.254

  external:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.4.0/24
          gateway: 192.168.4.254

  dns_net:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.2.0/24
          gateway: 192.168.2.254
```

还需要写一下dockerfile

client

```dockerfile
FROM kasmweb/ubuntu-focal-desktop:1.16.0

USER root

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y \
        wireshark \
        tcpdump \
        iproute2 \
        iputils-ping \
        net-tools \
        curl \
        vim \
        netcat-openbsd && \
    rm -rf /var/lib/apt/lists/*

```

dns

```dockerfile
FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y \
        iproute2 \
        iputils-ping \
        net-tools \
        scapy \
        tcpdump \
				vim \
        curl && \
    rm -rf /var/lib/apt/lists/*

COPY dnsmasq.conf /etc/dnsmasq.conf

```

server

```dockerfile
FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y \
        iproute2 \
        iputils-ping \
        net-tools \
        curl \
        nginx \
	vim \
        tcpdump && \
    rm -rf /var/lib/apt/lists/*

COPY index.html /var/www/html/index.html

```

访问浏览器

```
https://localhost:6901
kasm_user
password
```

然后访问

1. 在Client上先运行Wireshark，选择监听以太网网卡eth0的所有流量；

2. 在Client上通过浏览器访问 http://www.imool.com.cn网站，然后结束访问该网站；
3. ![image-20260911102029094](/Users/infinite/Library/Application Support/typora-user-images/image-20260911102029094.png)

3. 根据Wireshark中捕获到的流量，解释从开始访问www.imool.com.cn到结束访问整个过程中Client主机都发生了哪些网络活动，图1-2-2给出部分Wireshark捕获到的流量，特别关注：

（1）域名解析过程涉及哪些IP包，请求和响应分别是什么？

Client 首先向 DNS 服务器 192.168.2.53 发送针对 [www.imool.com.cn](http://www.imool.com.cn) 的 A 记录查询，DNS 服务器返回域名对应的 IPv4 地址 192.168.4.105。浏览器同时还发送了 Type 65（HTTPS Resource Record）查询，由于当前 DNS 服务器未提供该记录，因此返回 Refused，但不影响 A 记录解析及后续 HTTP 访问。

（2）ARP解析过程中，网关的MAC地址是什么？

**网关 IP：**192.168.3.1

**网关 MAC：** 1a:c0:cc:a9:51:49

（3）Client和www.imool.com.cn的连接建立过程、连接拆除过程；

![image-20260911103806399](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260911103806399.png)

获得服务器地址后，Client 使用临时端口 54338 与服务器 192.168.4.105 的 TCP 80 端口进行三次握手，即 SYN、SYN/ACK、ACK，建立 TCP 连接。随后 Client 发送 `GET / HTTP/1.1` 请求，服务器返回 HTTP 响应，抓包中可观察到 `HTTP/1.1 304 Not Modified`。Web 数据传输结束后，双方通过 FIN/ACK 报文完成 TCP 连接释放。

（4）使用Wireshark的协议流追踪功能，提取Web访问的Cookie信息

但是我们发现没有cookie，修改resolv.conf，访问真实的网页的话，就有了

![image-20260911105259873](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260911105259873.png)

## 实验三

再弄一个attacker

在compose中添加

```yaml
  attacker:
    build:
      context: ./attacker
    container_name: attacker
    hostname: attacker

    cap_add:
      - NET_ADMIN
      - NET_RAW

    depends_on:
      - route

    networks:
      dns_net:
        interface_name: eth0
        ipv4_address: 192.168.2.100

    command:
      - /bin/bash
      - -c
      - |
        ip route del default || true
        ip route add default via 192.168.2.1 dev eth0
```

然后dockerfile

```
FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y \
        iproute2 \
        iputils-ping \
        net-tools \
        scapy \
        tcpdump \
				vim \
        curl && \
    rm -rf /var/lib/apt/lists/*
```

部署启动attacker
```bash
 docker compose up -d attacker
```

打开scapy

![image-20260911111927185](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260911111927185.png)

首先创建IP层，将源ip（src）设置为User ip，将目的ip（dst）设置成Victim ip；

 ip_layer = IP(src="192.168.4.105", dst="192.168.3.10") 

3. 接下来，创建ICMP层，通过ls(ICMP)查看ICMP层默认字段值，其中type=8 和 code=0 组合起来表示 ICMP Echo Request（Ping） 数据包，未设置的字段将使用其默认值填充;

   ![image-20260911112056168](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260911112056168.png)

\>>> icmp_layer = ICMP() #创建ICMP层

​            4.     创建负载数据；

\>>> payload = "I come from Attacker-192.168.2.100"#载荷的内容

​            5.     然后将IP层、ICMP层、负载层三者进行组合，生成一个Ping数据包；

\>>> icmp_request = ip_layer / icmp_layer / payload

​            6.     可以通过.show()方法来查看已构造好的数据包内容；

![image-20260911112221253](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260911112221253.png)

发送

![image-20260911112317830](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260911112317830.png)

在client上面收到

![image-20260911112432347](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260911112432347.png)
