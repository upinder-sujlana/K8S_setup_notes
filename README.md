```
### Hardware is 3 VM

### Software
kmaster2@kmaster2:~$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 18.04 LTS
Release:        18.04
Codename:       bionic
kmaster2@kmaster2:~$

### IP address
192.168.1.86 - kmaster2 (control-plane,master node)
192.168.1.87 - knode3
192.168.1.88 - knode4


Step 1: On All Machines ( Master & All nodes ):
#----------------------------------------------------------------------
(OPTIONAL) echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
##REBOOT ALL MACHINES

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update ; clear
sudo apt-get install -y docker-ce

#Configure docker runtime to use systemd as the cgroup driver, same as kubelet.
#https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker
sudo mkdir /etc/docker  (if directory not there)
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker



# install  kubelet ,  kubeadm , kubectl
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-get update ; clear
sudo apt-get install -y kubelet kubeadm kubectl	

# Run below and verify that services are active
systemctl status kubelet
systemctl status docker


Step 2: On Master only:
#----------------------------------------------------------------------
sudo kubeadm init --ignore-preflight-errors=all
#Note the "kubeadm join" command output at the very end. Copy-paste in a notepad somewhere. We shall use it in Step 4
#e.g.
# kubeadm join 192.168.1.86:6443 --token adm81n.r5eiyf1njtm1mnxb \
        --discovery-token-ca-cert-hash sha256:170d455ca057576ed93fc670d9bb6592211cffcf06244fa91970c8398a278518


Step 3: On Master only:
#----------------------------------------------------------------------
sudo mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

## Deploy Weave for POD Network in the cluster. 
## For other network addons check https://kubernetes.io/docs/concepts/cluster-administration/addons/
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

Step 4 : On worker nodes only
#----------------------------------------------------------------------
# Run the join command noted from Step 2 from above.
# Add "sudo" in front and type password when prompted. e.g.
sudo kubeadm join 192.168.1.86:6443 --token adm81n.r5eiyf1njtm1mnxb \
        --discovery-token-ca-cert-hash sha256:170d455ca057576ed93fc670d9bb6592211cffcf06244fa91970c8398a278518

Step 5 : On master node
#----------------------------------------------------------------------
kmaster2@kmaster2:~$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   22m
kube-node-lease   Active   22m
kube-public       Active   22m
kube-system       Active   22m
kmaster2@kmaster2:~$ 

kmaster2@kmaster2:~$ kubectl get nodes
NAME       STATUS     ROLES                  AGE     VERSION
kmaster2   Ready      control-plane,master   10m     v1.22.2
knode3     Ready      <none>                 6m50s   v1.22.2
knode4     NotReady   <none>                 6m28s   v1.22.2
kmaster2@kmaster2:~$

Note:
--------
[+] I noticed that the knode4 was not ready, ran "journalctl -u kubelet" on knode4 and it said that having issues with CNI.
[+] On master I ran "kubectl get cs" and found scheduler had issues. I fixed it using the below 
    https://stackoverflow.com/questions/54608441/kubectl-connectivity-issue
    After that
    kmaster2@kmaster2:~$ kubectl get cs
    Warning: v1 ComponentStatus is deprecated in v1.19+
    NAME                 STATUS    MESSAGE                         ERROR
    scheduler            Healthy   ok
    controller-manager   Healthy   ok
    etcd-0               Healthy   {"health":"true","reason":""}
    kmaster2@kmaster2:~$

[+] Still the knode4 was having issues. 
[+] Decided to cleanup the knode4 (only)
kubeadm reset
sudo apt-get purge kubeadm kubectl kubelet kubernetes-cni kube*   
sudo apt-get autoremove  
sudo rm -rf ~/.kube
sudo reboot
[+] Did not have to cleanup docker, but if you need to cleanup docker as well use:
    https://askubuntu.com/questions/935569/how-to-completely-uninstall-docker
[+] Performed the re-install again of kubeadm kubectl kubelet
[+] Node came online

kmaster2@kmaster2:~$ kubectl get nodes
NAME       STATUS   ROLES                  AGE    VERSION
kmaster2   Ready    control-plane,master   3h1m   v1.22.2
knode3     Ready    <none>                 177m   v1.22.2
knode4     Ready    <none>                 176m   v1.22.2
kmaster2@kmaster2:~$



Step 6 : 
#----------------------------------------------------------------------
#Create a test jumpbox
kmaster2@kmaster2:~$ kubectl run multitool --image=praqma/network-multitool
pod/multitool created
kmaster2@kmaster2:~$
kmaster2@kmaster2:~$ kubectl get pods -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP          NODE     NOMINATED NODE   READINESS GATES
multitool   1/1     Running   0          33s   10.44.0.1   knode3   <none>           <none>
kmaster2@kmaster2:~$

Step 7 (Optional, if playing with etcd back etc) : 
#----------------------------------------------------------------------
ETCD_VER=v3.3.13

GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}

rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

/tmp/etcd-download-test/etcd --version
ETCDCTL_API=3 /tmp/etcd-download-test/etcdctl version

sudo mv /tmp/etcd-download-test/etcdctl /usr/bin


```
