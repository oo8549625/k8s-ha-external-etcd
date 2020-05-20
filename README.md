# 架設k8s + keepalived + haproxy

## ServerIP
| Server  | IP  |
| :------------ |:---------------:|
| master1 | 192.168.210.5 |
| master2 | 192.168.210.24 |
| master3 | 192.168.210.12 |
| VIP | 192.168.210.20 |
| node1 | 192.168.210.3 |
| node2 | 192.168.210.10 |
| node3 | 192.168.210.17 |

## 確定關閉所有防火牆

## 安裝, 配置及測試 keepalived
```
#流程
三台master安裝keepalived
cat >> /etc/sysctl.conf << EOF
net.ipv4.ip_forward = 1
EOF
sysctl -p
以下配置須修改(其餘一樣)
master1 state MASTER  priority 11 unicast_src_ip 192.168.210.5 unicast_peer {192.168.210.24 192.168.210.12}
master2 state BACKUP  priority 10 unicast_src_ip 192.168.210.24 unicast_peer {192.168.210.5 192.168.210.12}
master3 state BACKUP  priority 10 unicast_src_ip 192.168.210.12 unicast_peer {192.168.210.5 192.168.210.24}
配置完重起keepalived

#keepalived installation
sudo apt-get install -y keepalived

#開機執行
sudo systemctl enable keepalived

#配置文件
sudo nano /etc/keepalived/keepalived.conf

#刪除文件
sudo rm -rf /etc/keepalived/keepalived.conf

#重起｜查看狀態｜停止｜開始
sudo systemctl restart keepalived
sudo systemctl status keepalived
sudo systemctl stop keepalived
sudo systemctl start keepalived

# ip地址
sudo ip address

#測試
若是該master的STATE=MASTER, 查看ip地址會多出一個VIP, 此例子(192.168.210.20), 若為BACKUP則不會多出VIP
重開master1的keepalived之後, 看master1的keepalived狀態從MASTER-->BACKUP, priority高於master2但是不會搶佔
查看master2的keepalived狀態從BACKUP-->MASTER
接著, 重開master2的keepalived, master1回到MASTER STATE, master2 回到BACKUP STATE
```

## 安裝, 配置及測試 haproxy
```
#keepalived配置完成後,安裝haproxy
cat >> /etc/sysctl.conf << EOF
net.ipv4.ip_nonlocal_bind = 1
EOF
sysctl -p
三台master都要裝haproxy
三台master配置文件都一樣 VIP port 8443

#haproxy installation
sudo apt install haproxy -y

#開機執行
sudo systemctl enable haproxy

#配置文件
sudo nano /etc/haproxy/haproxy.cfg

#刪除文件
sudo rm -rf  /etc/haproxy/haproxy.cfg

#重起｜查看狀態｜停止｜開始
sudo systemctl restart haproxy
sudo systemctl status haproxy
sudo systemctl stop haproxy
sudo systemctl start haproxy

#查看ip
sudo netstat -lntp

#check ip port connect
nc -v {ip} {port}
nc -v 192.168.210.20 8443

#測試
首先確定master1 haproxy status work
連上後可以查看ip, 以配置可以看見 192.168.210.20:8443
每台都要連線測試, Connection to 192.168.210.20 8443 port [tcp/*] succeeded! , 表示連線成功
```

## 安裝, 配置及測試 k8s
```
#keepalived和haproxy配置完成後,跑流程
首先全部的node都跑k8s.sh,
master1 配置kubeadm init
master2 join control plane
master3 join control plane
node1 join node
node2 join node
node3 join node

#k8s.sh (script包含docker, helm, kubectl, kubeadm, kubelet 以及 ipvs)
sudo bash k8s.sh

#gitlab-runner installation(可選擇)
sudo apt install gitlab-runner -y
sudo usermod -aG gitlab-runner docker

#kubeadm configuration (script包含kubeadm init, gitlab-runner權限, pod網路, taint, helm repo add, admin clusterrole, kubelet control 權限)
#須將kudeadm.yaml一同放入同個dir
sudo bash kubeadm-conf.sh

#create cert (生成新的cert key)
kubeadm alpha certs certificate-key
 
#create token (生成新的token)
kubeadm token create --print-join-command

#kubelet configuration 
sudo cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

#join control-plane (BACKUP master node 加入)
kubeadm join 203.145.220.182:6443 --token uyx7zg.aaer3ibuc2bgucaq \
  --discovery-token-ca-cert-hash sha256:752530a95fc9bc66f7e54bac97bb04d66f79d347afbb2c6c351f95948a7742f8 \
  --control-plane --certificate-key e68b71cb71423092cb696f7caddf04bb3bce46300bb91f67f271dffe1167f31b

# join node (worker node 加入)
kubeadm join 203.145.220.182:6443 --token uyx7zg.aaer3ibuc2bgucaq \
  --discovery-token-ca-cert-hash sha256:752530a95fc9bc66f7e54bac97bb04d66f79d347afbb2c6c351f95948a7742f8 \


#測試
```


### Options
```
#調整kubernetes nodePort range
sudo nano /etc/kubernetes/manifests/kube-apiserver.yaml
command:
--service-node-port-range=1-65535

#update kubelet kubectl kubeadm version
sudo apt-mark unhold kubeadm kubectl kubelet
sudo apt-get update && apt-get install -y kubeadm kubelet kubectl
sudo apt-mark hold kubeadm kubectl kubelet
sudo kubeadm upgrade apply v1.18.x
sudo systemctl restart kubelet

#修正CSIM問題導致DNS pending與node notReady
sudo nano /var/lib/kubelet/config.yaml
featureGates:
  CSIMigration: false
sudo systemctl restart kubelet

#haproxy cannot bind socket
sudo nano  /etc/sysctl.conf
添加：net.ipv4.ip_nonlocal_bind=1

#master node join control plane error

#worker node join超時問題
sudo swapoff -a
sudo kubeadm reset  
sudo systemctl daemon-reload && sudo systemctl restart kubelet 
```
