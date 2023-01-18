# K8S-Step-by-Step-Installation
# k8s-installation-kubeadm
##  Pre-requisite 
### Hardware Specification:
*     Master nodes: 3 * ( 6 cpu - 16 GB memory - 70 GB Hard )
*     Worker node: 1 * ( 2 cpu - 8 GB memory - 150 + 100 GB Hard) 
      * For functional test one worker is enough for other cases more workder nodes may be needed
*     HA Nodes: 2 * ( 4 cpu - 6 GB memory - 50 GB Hard)
### OS Specification:
* Ubuntu 20.04
## Pre-installation steps -  The below steps will be performed on both master and worker nodes

### Diasble Swap 

*  1.  Turn of Swap

`   apt-get update`

`   swapoff -a ` 

*  2.  Comment swap FS from /etc/fstab 

`   vi /etc/fstab`

`   Comment any line that has swap written` 

### Check and edit Hostame if needed(espeially after clonning)

*  1.  Edit /etc/hostname and edit hostname to match the host of your choice 

`   vi /etc/hostname ` 

*  2.  Edit /etc/hosts to add hostname and IP address on all nodes 

`   vi /etc/hosts ` 

~~~

~~~
### Change Mac Identifier to DHCP
* 1. edit config file, e.g. as below 
` sudo vim /etc/netplan/01-netcfg.yaml`
* 2. add dhcp-identifier so config file looks like following
```yml
 network: 
    renderer: networkd
    version: 2
    ethernets:
        nicdevicename:
            dhcp4: true
            dhcp-identifier: mac 
```
* 3. apply new config
`   sudo netplan apply `
### Install VMware tools > opevmware tools (ADR)
```
mkdir /mnt/cdrom
mount /dev/cdrom /mnt/cdrom
ls /mnt/cdrom/
tar xzvf /mnt/cdrom/VMwareTools-10.3.24-18733423.tar.gz -C /tmp/
cd /tmp/vmware-tools-distrib/
./vmware-install.pl
```

## Installation Procedure 
### Installing a container runtime 
#### Install and configure prerequisites
##### Forwarding IPv4 and letting iptables see bridged traffic  ¶
~~~bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
 
sudo modprobe overlay
sudo modprobe br_netfilter
 
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
 
# Apply sysctl params without reboot
sudo sysctl --system
~~~
##### Cgroup drivers 
??? after containerd installation
#### Install and configure Containerd from apt-get or dnf
##### Uninstall old versions
` sudo apt-get remove docker docker-engine docker.io containerd runc`
##### Set up the repository
* Update the apt package index and install packages to allow apt to use a repository over HTTPS:
~~~bash
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
~~~
* Add Docker’s official GPG key:
~~~bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
~~~
* Use the following command to set up the repository:
~~~bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
~~~
##### Install Docker Engine
* Update the apt package index:
`sudo apt-get update`
* Receiving a GPG error when running apt-get update?
Your default umask may be incorrectly configured, preventing detection of the repository public key file. Try granting read permission for the Docker public key file before updating the package index:
~~~bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
sudo apt-get update
~~~


* Install Docker Engine, containerd, and Docker Compose.
`sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin`

* configure cgroup
~~~
cp -p /etc/containerd/config.toml /etc/containerd/config.BAK
vim /etc/containerd/config.toml
~~~
~~~yaml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
~~~
~~~
sudo systemctl daemon-reload
sudo systemctl restart containerd
~~~

### Install kubelet kubeadm and kubectl 
* 1. Update the apt package index and install packages needed to use the Kubernetes apt repository:
~~~
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
~~~
* 2. Download the Google Cloud public signing key:
`sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg `
* 3. Add the Kubernetes apt repository:
`echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list `
* 4. Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:
~~~bash
sudo apt-get update
sudo apt install -y kubeadm=1.23.8-00 kubelet=1.23.8-00 kubectl=1.23.8-00
sudo apt-mark hold kubelet kubeadm kubectl
~~~
## Creating Highly Available Clusters with kubeadm (Stacked control plane and etcd nodes)

## Set up load balancer nodes
### Install Keepalived & Haproxy
`apt update && apt install -y keepalived haproxy`
### configure keepalived
On both nodes create the health check script /etc/keepalived/check_apiserver.sh
~~~bash
cat >> /etc/keepalived/check_apiserver.sh <<EOF
#!/bin/sh

errorExit() {
  echo "*** $@" 1>&2
  exit 1
}

curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q 172.16.16.100; then
  curl --silent --max-time 2 --insecure https://172.16.16.100:6443/ -o /dev/null || errorExit "Error GET https://172.16.16.100:6443/"
fi
EOF

chmod +x /etc/keepalived/check_apiserver.sh
~~~
Create keepalived config /etc/keepalived/keepalived.conf

~~~bash
cat >> /etc/keepalived/keepalived.conf <<EOF
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  timeout 10
  fall 5
  rise 2
  weight -2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth1
    virtual_router_id 1
    priority 100
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass mysecret
    }
    virtual_ipaddress {
        172.16.16.100
    }
    track_script {
        check_apiserver
    }
}
EOF

~~~

### Enable & start keepalived service
`systemctl enable --now keepalived`
### Configure haproxy
Update /etc/haproxy/haproxy.cfg
~~~bash
cat >> /etc/haproxy/haproxy.cfg <<EOF

frontend kubernetes-frontend
  bind *:6443
  mode tcp
  option tcplog
  default_backend kubernetes-backend

backend kubernetes-backend
  option httpchk GET /healthz
  http-check expect status 200
  mode tcp
  option ssl-hello-chk
  balance roundrobin
    server kmaster1 172.16.16.101:6443 check fall 3 rise 2
    server kmaster2 172.16.16.102:6443 check fall 3 rise 2
    server kmaster3 172.16.16.103:6443 check fall 3 rise 2

EOF
~~~

### Enable & restart haproxy service
`systemctl enable haproxy && systemctl restart haproxy`

### Add the first control plane node to the load balancer, and test the connection
`nc -v <LOAD_BALANCER_IP> <PORT>`
### Add the remaining control plane nodes to the load balancer target group.






## Steps for the first control plane node 
* 1. Initialize the control plane:
~~~
sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs --pod-network-cidr=192.168.0.0/16 --image-repository <reponame>/k8s-offline
e.g.
kubeadm init --control-plane-endpoint="172.17.251.30:6443" --upload-certs --pod-network-cidr=192.168.0.0/16 --image-repository <reponame>/k8s-offline
~~~
* 2. Apply the CNI plugin of your choice
# cilium






####

#####




































