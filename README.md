# 架設k8s + external-etcd + haproxy

## ServerIP
| Server  | IP  |
| :------------ |:---------------:|
| haproxy | 192.168.210.3 |
| master1 | 192.168.210.4 |
| master2 | 192.168.210.5 |
| master3 | 192.168.210.6 |
| etcd1 | 192.168.210.7 |
| etcd2 | 192.168.210.8 |
| etcd3 | 192.168.210.9 |
| node1 | 192.168.210.10 |
| node2 | 192.168.210.11 |
| node3 | 192.168.210.12 |

## 確定關閉所有防火牆

## external etcd(docker, kubeadm, kubelet)

1. Configure the kubelet to be a service manager for etcd.
Since etcd was created first, you must override the service priority by creating a new unit file that has higher precedence than the kubeadm-provided kubelet unit file.(三台etcd都配置)

```
cat << EOF > /etc/systemd/system/kubelet.service.d/20-etcd-service-manager.conf
[Service]
ExecStart=
#  Replace "systemd" with the cgroup driver of your container runtime. The default value in the kubelet is "cgroupfs".
ExecStart=/usr/bin/kubelet --address=127.0.0.1 --pod-manifest-path=/etc/kubernetes/manifests --cgroup-driver=cgroupfs
Restart=always
EOF

systemctl daemon-reload
systemctl restart kubelet
```

2.Create configuration files for kubeadm.
Generate one kubeadm configuration file for each host that will have an etcd member running on it using the following script.

```
# Update HOST0, HOST1, and HOST2 with the IPs or resolvable names of your hosts
export HOST0=192.168.210.7
export HOST1=192.168.210.8
export HOST2=192.168.210.9

# Create temp directories to store files that will end up on other hosts.
mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/

ETCDHOSTS=(${HOST0} ${HOST1} ${HOST2})
NAMES=("etcd1" "etcd2" "etcd3")

for i in "${!ETCDHOSTS[@]}"; do
HOST=${ETCDHOSTS[$i]}
NAME=${NAMES[$i]}
cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
apiVersion: "kubeadm.k8s.io/v1beta2"
kind: ClusterConfiguration
etcd:
    local:
        serverCertSANs:
        - "${HOST}"
        peerCertSANs:
        - "${HOST}"
        extraArgs:
            initial-cluster: ${NAMES[0]}=https://${ETCDHOSTS[0]}:2380,${NAMES[1]}=https://${ETCDHOSTS[1]}:2380,${NAMES[2]}=https://${ETCDHOSTS[2]}:2380
            initial-cluster-state: new
            name: ${NAME}
            listen-peer-urls: https://${HOST}:2380
            listen-client-urls: https://${HOST}:2379
            advertise-client-urls: https://${HOST}:2379
            initial-advertise-peer-urls: https://${HOST}:2380
EOF
done
```

3.Generate the certificate authority

```
kubeadm init phase certs etcd-ca

This creates two files:
/etc/kubernetes/pki/etcd/ca.crt
/etc/kubernetes/pki/etcd/ca.key
```

4.Create certificates for each member

```
export HOST0=192.168.210.7
export HOST1=192.168.210.8
export HOST2=192.168.210.9

kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST2}/
# cleanup non-reusable certificates
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST1}/
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
# No need to move the certs because they are for HOST0

# clean up certs that should not be copied off this host
find /tmp/${HOST2} -name ca.key -type f -delete
find /tmp/${HOST1} -name ca.key -type f -delete
```

5. Copy certificates and kubeadm configs
The certificates have been generated and now they must be moved to their respective hosts

```
export HOST1=192.168.210.8
export HOST2=192.168.210.9

USER=ubuntu
scp -r /tmp/${HOST1}/* ${USER}@${HOST1}:
scp -r /tmp/${HOST2}/* ${USER}@${HOST2}:
```

6.Create the static pod manifests
Now that the certificates and configs are in place it’s time to create the manifests. On each host run the kubeadm command to generate a static manifest for etcd.

```
root@HOST0 $ kubeadm init phase etcd local --config=/tmp/${HOST0}/kubeadmcfg.yaml
root@HOST1 $ kubeadm init phase etcd local --config=/home/ubuntu/kubeadmcfg.yaml
root@HOST2 $ kubeadm init phase etcd local --config=/home/ubuntu/kubeadmcfg.yaml
```

7. Optional: Check the cluster health

```
docker run --rm -it \
--net host \
-v /etc/kubernetes:/etc/kubernetes k8s.gcr.io/etcd:${ETCD_TAG} etcdctl \
--cert /etc/kubernetes/pki/etcd/peer.crt \
--key /etc/kubernetes/pki/etcd/peer.key \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--endpoints https://${HOST0}:2379 endpoint health --cluster
...
https://[HOST0 IP]:2379 is healthy: successfully committed proposal: took = 16.283339ms
https://[HOST1 IP]:2379 is healthy: successfully committed proposal: took = 19.44402ms
https://[HOST2 IP]:2379 is healthy: successfully committed proposal: took = 35.926451ms
```

## 安裝, 配置及測試 haproxy
```

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

```

## 安裝, 配置及測試 k8s(docker, kubelet, kubeadm, kubectl, ipvs)

- 複製etcd的三個檔案到kubeadm init的control plane

```
export CONTROL_PLANE="ubuntu@192.168.210.4"
scp /etc/kubernetes/pki/etcd/ca.crt "${CONTROL_PLANE}":
scp /etc/kubernetes/pki/apiserver-etcd-client.crt "${CONTROL_PLANE}":
scp /etc/kubernetes/pki/apiserver-etcd-client.key "${CONTROL_PLANE}":
```

- master1 配置kubeadm init
- master2 join control plane
- master3 join control plane
- worker1 join node
- worker2 join node
- worker3 join node

```

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


```


### Options
```
#kubeadm join超時問題
sudo swapoff -a
sudo kubeadm reset  
sudo systemctl daemon-reload && sudo systemctl restart kubelet 

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

#Error haproxy cannot bind socket
sudo nano  /etc/sysctl.conf
添加：net.ipv4.ip_nonlocal_bind=1

#Error Throttling request took 1.177626024s, request: GET:https://192.168.210.21:6443/apis/coordination.k8s.io/v1?timeout=32s
#user ownership for kubectl
sudo chown -R $USER $HOME/.kube

#更換pod網路,刪除舊有cni
sudo rm -rf /etc/cni

#修改coreDNS replica
kubectl edit deployment.apps/coredns -n kube-system
replica 2-->6

# admin clusterrole
kubectl create clusterrolebinding <clusterrolebinding name> --clusterrole <clusterrole name> --serviceaccount=<namespace>:<name>
kubectl create clusterrolebinding default-admin --clusterrole cluster-admin --serviceaccount=default:default

#let master node work
#kubernetes出于安全考量默認情况下無法在master節點上部署pod, 以下指令允许master節點部署pod
kubectl taint nodes --all node-role.kubernetes.io/master-
```
