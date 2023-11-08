# MPLS标签入栈

```bash

ip netns exec sat4 tc qdisc replace dev sat4-ens2 ingress
sudo docker exec -it sat4 /bin/bash -c 'tc qdisc add dev sat4-ens2 clsact';

# a4

ip netns exec sat4 tc filter replace dev sat4-ens2 ingress protocol ip \
flower ip_proto tcp dst_ip 152.0.91.1/32 \
src_ip 152.0.81.1/32 ip_tos 0xa4 \
action mpls push protocol mpls_uc tc 2 label 17001 \
action mpls push protocol mpls_uc tc 2 label 17002 \
action mpls push protocol mpls_uc tc 2 label 17003


ip netns exec sat1 tc qdisc replace dev sat1-ens ingress
sudo docker exec -it sat1 /bin/bash -c 'tc qdisc add dev sat1-ens clsact';

ip netns exec sat1 tc filter replace dev sat1-ens ingress protocol ip \
flower ip_proto tcp dst_ip 152.0.81.1/32 \
src_ip 152.0.91.1/32 ip_tos 0x00 \
action mpls push protocol mpls_uc tc 2 label 17004 \
action mpls push protocol mpls_uc tc 2 label 17003 \
action mpls push protocol mpls_uc tc 2 label 17002

```



```bash
sudo docker exec -it sat1 /bin/bash -c 'ip route rep 192.168.1.4/32 via 152.0.12.2’
sudo docker exec -it sat4 /bin/bash -c 'ip route rep 192.168.1.1/32 via 152.0.43.2’



ip netns exec sat3 tc qdisc replace dev sat3-sat4 ingress
sudo docker exec -it sat3 /bin/bash -c 'tc qdisc add dev sat3-sat4 clsact';

ip netns exec sat3 tc filter replace dev sat3-sat4 ingress protocol ip \
flower ip_proto tcp dst_ip 192.168.1.1/32 \
src_ip 192.168.1.4/32 \
action mpls push protocol mpls_uc tc 2 label 17001 \
action mpls push protocol mpls_uc tc 2 label 17002 


ip netns exec sat2 tc qdisc replace dev sat2-sat1 ingress
sudo docker exec -it sat2 /bin/bash -c 'tc qdisc add dev sat2-sat1 clsact';

ip netns exec sat2 tc filter replace dev sat2-sat1 ingress protocol ip \
flower ip_proto tcp dst_ip 192.168.1.4/32 \
src_ip 192.168.1.1/32 \
action mpls push protocol mpls_uc tc 2 label 17004 \
action mpls push protocol mpls_uc tc 2 label 17003 


ip netns exec sat3 tc filter del dev sat3-sat4 ingress
ip netns exec sat2 tc filter del dev sat2-sat1 ingress

```





```bash
sudo docker exec -it sat1 /bin/bash -c 'ip route rep 192.168.1.4/32 via 152.0.12.2'
sudo docker exec -it sat4 /bin/bash -c 'ip route rep 192.168.1.1/32 via 152.0.43.2'


ip netns exec sat2 tc qdisc replace dev sat2-sat1 ingress
sudo docker exec -it sat2 /bin/bash -c 'tc qdisc add dev sat2-sat1 clsact';
ip netns exec sat2 tc filter replace dev sat2-sat1 ingress protocol ip \
flower ip_proto tcp dst_ip 192.168.1.4/32 \
src_ip 192.168.1.1/32 \
action mpls push protocol mpls_uc label 17004 \
action mpls push protocol mpls_uc label 17003 \
action mpls push protocol mpls_uc label 17002 \
action skbedit priority 1 


ip netns exec sat3 tc qdisc replace dev sat3-sat4 ingress
sudo docker exec -it sat3 /bin/bash -c 'tc qdisc add dev sat3-sat4 clsact';
ip netns exec sat3 tc filter replace dev sat3-sat4 ingress protocol ip \
flower ip_proto tcp dst_ip 192.168.1.1/32 \
src_ip 192.168.1.4/32 \
action mpls push protocol mpls_uc label 17001 \
action mpls push protocol mpls_uc label 17002 \
action mpls push protocol mpls_uc label 17003 \
action skbedit priority 1 

```

