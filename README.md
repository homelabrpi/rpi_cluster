# rpi_cluster
## create kubernetes cluster using raspberry pis

devices:
- rpi (rpi4b - with raspberry pi os lite 64bit) (32 Gb memory card)
- rpi (rpi3b - with raspberry pi os lite 64bit) (32 Gb memory card)
- tplink router + network switch


## First steps: 
- Create an empty file named ssh (without any extension) in the boot partition of the SD card to enable SSH.
- create a file called `userconf` in the same boot partition of the SD card.  This file should contain a single line of text, consisting of `{name}:{encrypted-password}` ( I used node for my login user but use what you want.)
- To generate the encrypted-password, run the following command with OpenSSL: `echo '{password}' | openssl passwd -6 -stdin`

## First Boot and Initial Configuration:
Setup static ip for the pies (router config) (Create static ip for the pies -> easy to remember)
Find the mac adress of the pi -> DHCP -> address reservation
```
raspberrypi4 - 192.168.0.100
raspberrypi3 - 192.168.0.150
```

SSH to the first node (CP)
(userconf not worked to me so manually create node user)
```
sudo adduser node
sudo usermod -aG sudo,adm,dialout,cdrom,plugdev,users,input,audio,video,node node
```

Set Up SSH for the node User
```
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

Add your user to the sudo group with the following command.
```
sudo usermod -aG sudo node
```

Now lets update the rasp-config to autoboot with the node user. (System options -> Boot / autologin -> console autologin)
```
sudo raspi-config
```

### Docker & Kubernetes Initial Set Up
By default the cgroup memory option will be disabled, we will need to update this for Docker to be able to limit memory usage. Open /boot/firmware/cmdline.txt and append `cgroup_enable=memory cgroup_memory=1`.

```
echo " cgroup_enable=memory cgroup_memory=1" | sudo tee -a /boot/firmware/cmdline.txt
```

Now lets update our apt repository and include the Kubernetes repository. (https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

(sudo systemctl enable --now kubelet)
```

Docker install:
```
curl -sSL https://get.docker.com | sh
sudo usermod -aG docker node
```

To install cri-dockerd and set up the service, run the following commands:
```
sudo apt update
sudo apt install -y git golang-go make
#git clone https://github.com/Mirantis/cri-dockerd.git
#cd cri-dockerd

wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.16/cri-dockerd-0.3.16.arm64.tgz
tar -xvzf cri-dockerd-0.3.16.arm64.tgz
sudo mv cri-dockerd/cri-dockerd /usr/bin/cri-dockerd
sudo chmod +x /usr/bin/cri-dockerd
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
sudo mv cri-docker.service /etc/systemd/system/
sudo mv cri-docker.socket /etc/systemd/system/
sudo systemctl enable cri-docker.service
sudo systemctl enable cri-docker.socket
sudo systemctl start cri-docker.service
sudo systemctl start cri-docker.socket
```

It’s recommended to disable swap on our nodes for the Kubernetes scheduler.
```
sudo apt-get update && sudo apt-get install dphys-swapfile && sudo dphys-swapfile 
swapoff -a && sudo dphys-swapfile uninstall && sudo systemctl disable dphys-swapfile
```

## Initialize the cluster (CONTROL plane)

If you want to generate a new token dynamically, you can use:
```
kubeadm token generate > token

kubeadm token create --print-join-command
kubeadm token list
```

```
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: <generated token>
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.0.100
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/cri-dockerd.sock
  imagePullPolicy: IfNotPresent
  name: control-plane
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  podSubnet: "10.244.0.0/16" # --pod-network-cidr
```

```
kubeadm config migrate --old-config kubeadm-config.yaml --new-config kubeadm-config-v1.yaml
```


```
sudo hostnamectl set-hostname control-plane
# Update /etc/hosts to map the hostname to 127.0.0.1:
sudo nano /etc/hosts  # 127.0.0.1   control-plane
```

`sudo kubeadm init --config kubeadm-config-v1.yaml`

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

or
`export KUBECONFIG=/etc/kubernetes/admin.conf`

`sudo systemctl restart kubelet`
```
sudo reboot
```

You should now deploy a pod network to the cluster. (we will use flannel)
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

```
vim /run/flannel/subnet.env

paste this:
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true


sudo modprobe br_netfilter
```

After this -> the control plane started to work

## Adding New Nodes To The Cluster

(similar setup k8s, docker)

```

```
sudo hostnamectl set-hostname node0
# Update /etc/hosts to map the hostname to 127.0.0.1:
sudo nano /etc/hosts  # 127.0.0.1   node0
```
```

Intialize worker node  (token stored on the cp) ()

get token: (run cmd on the control plane)
```
sudo kubeadm token list | awk 'NR == 2 {print $1}'
```

create token on the cp
```
kubeadm token create --print-join-command

```

```
sudo kubeadm join 192.168.0.100:6443 --token <token> --discovery-token-ca-cert-hash <hash> --cri-socket unix:///var/run/cri-dockerd.sock --node-name node0


vim /run/flannel/subnet.env

paste this:
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true


sudo modprobe br_netfilter
```
```
sudo systemctl restart kubelet
```

```
sudo ip link set cni0 down && sudo ip link set flannel.1 down
sudo ip link delete cni0 && sudo ip link delete flannel.1
systemctl restart containerd && systemctl restart kubelet


```

