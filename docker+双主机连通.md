## Docker通信

###1.1建立卫星容器环境

```bash
# 容器环境建立
docker rm -f $(docker ps -aq)

systemctl restart docker

mkdir -p /var/run/netns

sudo modprobe mpls_router

sudo modprobe mpls_gso

sudo modprobe mpls_iptunnel

for satIndex in 1 2 3 4
do
	docker run -itd --name sat$satIndex --network none --hostname sat$satIndex -v /etc/localtime:/etc/localtime:ro -v /home/sat$satIndex:/etc/frr --privileged lemonround/satellite:v1.0
done

sat1_pid=`docker inspect -f '{{.State.Pid}}' sat1`
#3742

sat2_pid=`docker inspect -f '{{.State.Pid}}' sat2`
#3623

sat3_pid=`docker inspect -f '{{.State.Pid}}' sat3`
#3495

sat4_pid=`docker inspect -f '{{.State.Pid}}' sat4`
#3419

#sat1:
ln -s /proc/$sat1_pid/ns/net /var/run/netns/sat1

#sat2:
#pid:2312
ln -s /proc/$sat2_pid/ns/net /var/run/netns/sat2

#sat3:
#pid:2205
ln -s /proc/$sat3_pid/ns/net /var/run/netns/sat3

#sat4:
ln -s /proc/$sat4_pid/ns/net /var/run/netns/sat4


#3. 创建veth-pair


### 创建veth-pair

###卫星和卫星
############################################################
ip link add sat1-sat2 numrxqueues 16 numtxqueues 16 type veth peer name sat2-sat1 numrxqueues 16 numtxqueues 16 

# 配置sat1-sat2
ip link set dev sat1-sat2 up
ip link set dev sat1-sat2 netns sat1

ip netns exec sat1 ip link set dev sat1-sat2 up
ip netns exec sat1 ip addr add dev sat1-sat2 152.0.12.1/24

# 配置sat2-sat1
ip link set dev sat2-sat1 up
ip link set dev sat2-sat1 netns sat2

ip netns exec sat2 ip link set dev sat2-sat1 up
ip netns exec sat2 ip addr add dev sat2-sat1 152.0.12.2/24

############################################################
ip link add sat2-sat3 numrxqueues 16 numtxqueues 16 type veth peer name sat3-sat2 numrxqueues 16 numtxqueues 16
# 配置sat2-sat3
ip link set dev sat2-sat3 up
ip link set dev sat2-sat3 netns sat2

ip netns exec sat2 ip link set dev sat2-sat3 up
ip netns exec sat2 ip addr add dev sat2-sat3 152.0.23.1/24

# 配置sat3-sat2
ip link set dev sat3-sat2 up
ip link set dev sat3-sat2 netns sat3

ip netns exec sat3 ip link set dev sat3-sat2 up
ip netns exec sat3 ip addr add dev sat3-sat2 152.0.23.2/24


############################################################
ip link add sat3-sat1 numrxqueues 16 numtxqueues 16 type veth peer name sat1-sat3 numrxqueues 16 numtxqueues 16
# 配置sat3-sat1
ip link set dev sat3-sat1 up
ip link set dev sat3-sat1 netns sat3

ip netns exec sat3 ip link set dev sat3-sat1 up
ip netns exec sat3 ip addr add dev sat3-sat1 152.0.31.1/24

# 配置sat1-sat3
ip link set dev sat1-sat3 up
ip link set dev sat1-sat3 netns sat1

ip netns exec sat1 ip link set dev sat1-sat3 up
ip netns exec sat1 ip addr add dev sat1-sat3 152.0.31.2/24


############################################################
ip link add sat4-sat2 numrxqueues 16 numtxqueues 16 type veth peer name sat2-sat4 numrxqueues 16 numtxqueues 16
# 配置sat4-sat2
ip link set dev sat4-sat2 up
ip link set dev sat4-sat2 netns sat4

ip netns exec sat4 ip link set dev sat4-sat2 up
ip netns exec sat4 ip addr add dev sat4-sat2 152.0.42.1/24

# 配置sat2-sat4
ip link set dev sat2-sat4 up
ip link set dev sat2-sat4 netns sat2

ip netns exec sat2 ip link set dev sat2-sat4 up
ip netns exec sat2 ip addr add dev sat2-sat4 152.0.42.2/24



############################################################
ip link add sat4-sat3 numrxqueues 16 numtxqueues 16 type veth peer name sat3-sat4 numrxqueues 16 numtxqueues 16
# 配置sat4-sat3
ip link set dev sat4-sat3 up
ip link set dev sat4-sat3 netns sat4

ip netns exec sat4 ip link set dev sat4-sat3 up
ip netns exec sat4 ip addr add dev sat4-sat3 152.0.43.1/24

# 配置sat3-sat4
ip link set dev sat3-sat4 up
ip link set dev sat3-sat4 netns sat3

ip netns exec sat3 ip link set dev sat3-sat4 up
ip netns exec sat3 ip addr add dev sat3-sat4 152.0.43.2/24





# 4. 进入容器开启相关模块

# 进入到容器中
# tip:必须得给每个卫星的链路开启mpls转发，不然转发不了呀
# ps.容器的模块继承虚拟机，在虚拟机中开了mpls转发模块就行，容器中也有了
# sat4
sudo docker exec -it sat4 /bin/bash -c 'sysctl -w net.ipv4.ip_forward=1;sysctl -w net.ipv4.conf.all.rp_filter=0;sysctl -w net.ipv4.conf.lo.rp_filter=0;sysctl -w net.mpls.conf.lo.input=1;sysctl -w net.ipv4.conf.sat4-sat2.rp_filter=0;sysctl -w net.mpls.conf.sat4-sat2.input=1;sysctl -w net.ipv4.conf.sat4-sat3.rp_filter=0;sysctl -w net.mpls.conf.sat4-sat3.input=1;sysctl -w net.mpls.platform_labels=1048575'

sudo docker exec -it sat4 /bin/bash -c 'echo 1 MPLS >> /etc/iproute2/rt_tables'

#sat3
sudo docker exec -it sat3 /bin/bash -c 'sysctl -w net.ipv4.ip_forward=1;sysctl -w net.ipv4.conf.all.rp_filter=0;sysctl -w net.ipv4.conf.lo.rp_filter=0;sysctl -w net.mpls.conf.lo.input=1;sysctl -w net.ipv4.conf.sat3-sat1.rp_filter=0;sysctl -w net.mpls.conf.sat3-sat1.input=1;sysctl -w net.ipv4.conf.sat3-sat2.rp_filter=0;sysctl -w net.mpls.conf.sat3-sat2.input=1;sysctl -w net.ipv4.conf.sat3-sat4.rp_filter=0;sysctl -w net.mpls.conf.sat3-sat4.input=1;sysctl -w net.mpls.platform_labels=1048575'

sudo docker exec -it sat3 /bin/bash -c 'echo 1 MPLS >> /etc/iproute2/rt_tables'

#sat2
sudo docker exec -it sat2 /bin/bash -c 'sysctl -w net.ipv4.ip_forward=1;sysctl -w net.ipv4.conf.all.rp_filter=0;sysctl -w net.ipv4.conf.lo.rp_filter=0;sysctl -w net.mpls.conf.lo.input=1;sysctl -w net.ipv4.conf.sat2-sat1.rp_filter=0;sysctl -w net.mpls.conf.sat2-sat1.input=1;sysctl -w net.ipv4.conf.sat2-sat3.rp_filter=0;sysctl -w net.mpls.conf.sat2-sat3.input=1;sysctl -w net.ipv4.conf.sat2-sat4.rp_filter=0;sysctl -w net.mpls.conf.sat2-sat4.input=1;sysctl -w net.mpls.platform_labels=1048575'

sudo docker exec -it sat2 /bin/bash -c 'echo 1 MPLS >> /etc/iproute2/rt_tables'

#sat1
sudo docker exec -it sat1 /bin/bash -c 'sysctl -w net.ipv4.ip_forward=1;sysctl -w net.ipv4.conf.all.rp_filter=0;sysctl -w net.ipv4.conf.lo.rp_filter=0;sysctl -w net.mpls.conf.lo.input=1;sysctl -w net.ipv4.conf.sat1-sat2.rp_filter=0;sysctl -w net.mpls.conf.sat1-sat2.input=1;sysctl -w net.ipv4.conf.sat1-sat3.rp_filter=0;sysctl -w net.mpls.conf.sat1-sat3.input=1;sysctl -w net.mpls.platform_labels=1048575'

sudo docker exec -it sat1 /bin/bash -c 'echo 1 MPLS >> /etc/iproute2/rt_tables'


```

###1.2建立SRS与ffmpeg容器

```bash
docker network create --subnet=172.18.0.0/16 --gateway=172.18.0.1 srs_ffmpeg_network

# 打开ffmpeg容器和SRS服务器容器
# SRS服务器
docker  run -itd --name=ubuntu-srs --network=srs_ffmpeg_network --ip=172.18.0.2 alexkkbupt/ubuntu-srs-alexk:v0.1
# 进入容器
docker exec -it --privileged ubuntu-srs bash
# 进入目录
cd srs/trunk/
# 启动SRS
./objs/srs -c conf/srs.conf
# 查看SRS的状态
./etc/init.d/srs status
# 设置容器命名空间
ubuntu_srs_pid=`docker inspect -f '{{.State.Pid}}' ubuntu-srs`
# 3632
ln -s /proc/$ubuntu_srs_pid/ns/net /var/run/netns/ubuntu-srs

# ffmpeg服务器

docker  run -itd --name=ubuntu-ffmpeg --network=srs_ffmpeg_network --ip=172.18.0.3 alexkkbupt/ubuntu-ffmpeg-alexk:v0.1
# 进入容器
docker exec -it --privileged ubuntu-ffmpeg bash
# 设置容器命名空间
ubuntu_ffmpeg_pid=`docker inspect -f '{{.State.Pid}}' ubuntu-ffmpeg`
# 3780
ln -s /proc/$ubuntu_ffmpeg_pid/ns/net /var/run/netns/ubuntu-ffmpeg

```



###1.3搭建SRS与ffmpeg容器链路

```shell
ip link add ens-sat1 numrxqueues 16 numtxqueues 16 type veth peer name sat1-ens numrxqueues 16 numtxqueues 16 

# 配置ens-sat1
ip link set dev ens-sat1 up
ip link set dev ens-sat1 netns ubuntu-srs

ip netns exec ubuntu-srs ip link set dev ens-sat1 up
ip netns exec ubuntu-srs ip addr add dev ens-sat1 152.0.91.1/24

# 配置ens-sat1
ip link set dev sat1-ens up
ip link set dev sat1-ens netns sat1

ip netns exec sat1 ip link set dev sat1-ens up
ip netns exec sat1 ip addr add dev sat1-ens 152.0.91.2/24

# 添加SNAT（源地址转换）规则
# ip netns exec sat1 iptables -t nat -A POSTROUTING -s 152.0.81.2 -j SNAT --to-source 152.0.81.1

# 添加SNAT（源地址转换）规则
# ip netns exec sat1 iptables -t nat -A POSTROUTING -s 152.0.91.1 -j SNAT --to-source 152.0.91.2

# 添加DNAT（目的地址转换）规则
# ip netns exec sat1 iptables -t nat -A PREROUTING -d 152.0.81.1 -j DNAT --to-destination 152.0.81.2

# 添加DNAT（目的地址转换）规则
# ip netns exec sat1 iptables -t nat -A PREROUTING -d 152.0.91.2 -j DNAT --to-destination 152.0.91.1


ip link add ens2-sat4 numrxqueues 16 numtxqueues 16 type veth peer name sat4-ens2 numrxqueues 16 numtxqueues 16 

# 配置ens-sat4
ip link set dev ens2-sat4 up
ip link set dev ens2-sat4 netns ubuntu-ffmpeg

ip netns exec ubuntu-ffmpeg ip link set dev ens2-sat4 up
ip netns exec ubuntu-ffmpeg ip addr add dev ens2-sat4 152.0.81.1/24


# 配置ens-sat4
ip link set dev sat4-ens2 up
ip link set dev sat4-ens2 netns sat4

ip netns exec sat4 ip link set dev sat4-ens2 up
ip netns exec sat4 ip addr add dev sat4-ens2 152.0.81.2/24

# 添加SNAT（源地址转换）规则
# ip netns exec sat4 iptables -t nat -A POSTROUTING -s 152.0.91.2 -j SNAT --to-source 152.0.91.1

# 添加SNAT（源地址转换）规则
# ip netns exec sat4 iptables -t nat -A POSTROUTING -s 152.0.81.1 -j SNAT --to-source 152.0.81.2

# 添加DNAT（目的地址转换）规则
# ip netns exec sat4 iptables -t nat -A PREROUTING -d 152.0.91.1 -j DNAT --to-destination 152.0.91.2

# 添加DNAT（目的地址转换）规则
# ip netns exec sat4 iptables -t nat -A PREROUTING -d 152.0.81.2 -j DNAT --to-destination 152.0.81.1

# ip route add 152.0.81.1 via 152.0.91.2
# ip route add 152.0.91.1 via 152.0.81.2

```

###1.4设置SRS与ffmpeg路由规则与DSCP

```bash
# SRS服务器
docker exec -it --privileged ubuntu-srs bash
# 创建网卡后执行
# 添加路由
ip route rep 0.0.0.0/0 via 152.0.91.2
ip route rep 172.18.0.0/16 via 172.18.0.2

	
ip netns exec ubuntu-srs tc qdisc rep dev ens-sat1 root netem delay 2000ms

# ffmpeg容器
# 创建网卡后执行
# 添加路由
docker exec -it --privileged ubuntu-ffmpeg bash
ip route rep 0.0.0.0/0 via 152.0.81.2
ip route rep 172.18.0.0/16 via 172.18.0.3

iptables -t mangle -A POSTROUTING -p TCP --dport 1935 -j DSCP --set-dscp 40



# 宿主机上的操作
 ip route add 152.0.91.1 via 172.18.0.2
```

###1.5推流

```bash
# 进入容器
docker exec -it --privileged ubuntu-ffmpeg bash
# 进入目录
cd ffmpeg
ffmpeg -re -i ./doc/source.flv -c copy -f flv rtmp://152.0.91.1/live/livestream
```



## 2.抓包分析

```bash
# 抓包
ip netns exec ubuntu-ffmpeg tcpdump -i ens2-sat4 -A -w /home/sunkk/Documents/Wireshark/VNET_MPLS/ens2-sat4.cap &
ip netns exec ubuntu-srs tcpdump -i ens-sat1 -A -w /home/sunkk/Documents/Wireshark/VNET_MPLS/ens-sat1.cap &
ip netns exec sat1 tcpdump -i sat1-ens -A -w /home/sunkk/Documents/Wireshark/VNET_MPLS/sat1-ens.cap &
ip netns exec sat1 tcpdump -i sat1-sat3 -A -w /home/sunkk/Documents/Wireshark/VNET_MPLS/sat1-sat3.cap &
ip netns exec sat1 tcpdump -i sat1-sat2 -A -w /home/sunkk/Documents/Wireshark/VNET_MPLS/sat1-sat2.cap &
ip netns exec sat4 tcpdump -i sat4-sat3 -A -w /home/sunkk/Documents/Wireshark/VNET_MPLS/sat4-sat3.cap &
ip netns exec sat4 tcpdump -i sat4-sat2 -A -w /home/sunkk/Documents/Wireshark/VNET_MPLS/sat4-sat2.cap &
ip netns exec sat4 tcpdump -i sat4-ens2 -A -w /home/sunkk/Documents/Wireshark/VNET_MPLS/sat4-ens2.cap &
sleep 2
killall tcpdump
```





##在五链四表的哪个位置设置DSCP

```bash
在 iptables 中设置 DSCP 标记，应选择正确的链和表。PREROUTING 链用于对进入本机的数据包进行处理。而对于本机产生并发送出去的数据包，您应当使用 POSTROUTING 链，或者在 OUTPUT 链中进行设置。这是因为 PREROUTING 链影响的是进入系统的包，而不是离开系统的包。
```















## 三PC通信

```bash
# 主机A：172.16.179.164

ip route add 172.16.179.165 via 172.16.179.156
```



```bash
# docekr主机：172.16.179.156
# 与主机A相连的网卡152.0.91.1 ens-sat1
# 与主机B相连的网卡152.0.81.1 ens2-sat4
# 开启转发

sysctl -w net.ipv4.ip_forward=1

sudo ip addr add 172.16.179.155/24 dev ens33 label ens32

# ip route add 172.16.179.165 via 152.0.91.1

iptables -t nat -A PREROUTING -i ens33 -d 172.16.179.165 -j DNAT --to-destination 152.0.81.1

# iptables -t nat -A PREROUTING -i ens-sat1 -d 172.16.179.165 -j DNAT --to-destination 152.0.81.1


iptables -t nat -A PREROUTING -i ens2-sat4 -d 152.0.81.1 -j DNAT --to-destination 172.16.179.165

```



```bash
主机B：172.16.179.165

ip route add 172.16.179.164 via 172.16.179.156

```

