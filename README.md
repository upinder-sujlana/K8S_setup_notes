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

sudo vi /etc/docker/daemon.json
##Copy paste the below into the file
{
		"exec-opts": ["native.cgroupdriver=systemd"]
}

sudo service docker restart


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

## Deploy Weave for POD Network in the cluster. For other network addons check https://kubernetes.io/docs/concepts/cluster-administration/addons/
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

Step 4 : On worker nodes only
#----------------------------------------------------------------------
# run the join command noted from Step 2 from above. Add "sudo" in front and type password when prompted
sudo kubeadm join 192.168.1.86:6443 --token adm81n.r5eiyf1njtm1mnxb \
        --discovery-token-ca-cert-hash sha256:170d455ca057576ed93fc670d9bb6592211cffcf06244fa91970c8398a278518

Step 5 : On master node
#----------------------------------------------------------------------
kmaster2@kmaster2:~$ kubectl get nodes
NAME       STATUS     ROLES                  AGE     VERSION
kmaster2   Ready      control-plane,master   10m     v1.22.2
knode3     Ready      <none>                 6m50s   v1.22.2
knode4     NotReady   <none>                 6m28s   v1.22.2
kmaster2@kmaster2:~$
kmaster2@kmaster2:~$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   22m
kube-node-lease   Active   22m
kube-public       Active   22m
kube-system       Active   22m
kmaster2@kmaster2:~$ 

Step 6 : 
kmaster2@kmaster2:~$ kubectl run multitool --image=praqma/network-multitool
pod/multitool created
kmaster2@kmaster2:~$
kmaster2@kmaster2:~$ kubectl get pods -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP          NODE     NOMINATED NODE   READINESS GATES
multitool   1/1     Running   0          33s   10.44.0.1   knode3   <none>           <none>
kmaster2@kmaster2:~$


```
