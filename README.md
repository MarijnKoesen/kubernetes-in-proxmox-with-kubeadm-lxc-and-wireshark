# Installing kubernetes master on proxmox behind NAT using wireshark

**NOTE: These are my personal notes on what I needed to do to get kubernetes working in proxmox using LXC and a container, they may not be complete/work on your machine, PR's are welcome**

```
# <- execute inside the (proxmox) host
$ <- execite inside the container
```



## Step 1: Prepare the proxmox host

Ensure the following modules are loaded:

```
# cat /proc/sys/net/bridge/bridge-nf-call-iptables
```

Now make sure swapiness is on 0, so that swap will not be used, otherwise kubernetes will not start:

```
# cat /proc/sys/vm/swappiness
[should be 0]
```

Define the new one

```
# sysctl vm.swappiness=0
```

Disable SWAP, it'll take some times to clean the SWAP area
```
#s wapoff -a
```

Now wait for swap to be empty.




## Step 1: Creating the kubernetes container

1) Create a new container in proxmox, making sure to give it 0 swap, and make it a privileged container
2) Edit the config file `/etc/pve/lxc/$ID.conf` and add the following part:

```
lxc.apparmor.profile: unconfined
lxc.cgroup.devices.allow: a
lxc.cap.drop:
lxc.mount.auto: "proc:rw sys:rw"
```

If you are using zfs on proxmos, make sure to create a ext4 volume, as zfs is not supported with kubeadm
See: https://github.com/corneliusweig/kubernetes-lxd

```
zfs create -V 50G mypool/my-dockervol
zfs create -V 5G mypool/my-kubeletvol
mkfs.ext4 /dev/zvol/mypool/my-dockervol
mkfs.ext4 /dev/zvol/mypool/my-kubeletvol
```

Then make sure to mount it inside of the container:

```
mp0: /dev/zvol/mypool/my-dockervol,mp=/var/lib/docker,backup=0
mp1: /dev/zvol/mypool/my-kubeletvol,mp=/var/lib/kubelet,backup=0
```


Next make sure `conntrack` is working in the container

```
$ sudo conntrack -L
```

Now we can setup the VPN we need, see documentation for wireguard to install

```
$ sudo add-apt-repository ppa:wireguard/wireguard
$ sudo apt-get update
$ sudo apt-get install wireguard
```

Create the config:

```
$ cat > /etc/wireguard/wg0.conf
[Interface]
Address = 10.0.0.1/32
ListenPort = 55555
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
PrivateKey = WNeaIBT40mN/asu9zXrPeSYA+4pFmZA9lUBvHTx+TG8=

[Peer]
# server2
PublicKey = NSWzZOIUHPqRxOxUmB/A7+Gs6oECYGojREvGs/ZEi2o=
AllowedIPs = 10.0.0.2/32

[Peer]
# server3
PublicKey = JhT41so2SiITMe2uqPoNB40kkwxRqyklWiILyhT1uVY=
AllowedIPs = 10.0.0.3/32
```

And start the vpn:

```
$ wg-quick up wg0
$ wg show
```

To make sure we start the vpn on boot, and to fix some other small issues create the following rc.local file:

```
$ cat > /etc/rc.local
#!/bin/sh -e

# Kubeadm 1.15 needs /dev/kmsg to be there, but it's not in lxc, but we can just use /dev/console instead
# see: https://github.com/kubernetes-sigs/kind/issues/662
if [ ! -e /dev/kmsg ]; then
    ln -s /dev/console /dev/kmsg
fi

# Make sure our VPN is setup so we can connect to the other nodes
wg-quick up wg0

# https://medium.com/@kvaps/run-kubernetes-in-lxc-container-f04aa94b6c9c
mount --make-rshared /' > /etc/rc.local

exit 0
```

Set the permissions and reboot

```
$ chmod +x /etc/rc.local
# sudo reboot
```

Now make sure we have booted properly:

```
$ ls -l /dev/kmsg
[this should exist]
$ wg show
[this should show wireguard is started]
```


Now we can start installing kubernetes:

```
$ apt-get update && apt-get install -y apt-transport-https curl
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
$ cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
$ apt-get update
$ apt-get install -y kubelet kubeadm kubectl
$ apt-mark hold kubelet kubeadm kubectl
```

To make sure kubernets connects using the correct ip, and will use the VPN to connect though, we must tell kubelet to use the vpn server ip:

```
$ echo "KUBELET_EXTRA_ARGS=--node-ip=10.0.0.1" >> /etc/default/kubelet
```

Now we can setup kubeadm:

Make sure to specify the pod-network-cidr and service-cidr if they are overlapping with your normal LAN network, which it is in my case, as I run 192.168.x.x on my proxmox LAN. Note that I add an extra apiserver-cert-extra-sans, so I can just connect to the api server from it's LAN ip too.

To make sure it will work we need to ignore all preflight errors, but it should work just fine.


```
$ kubeadm init --pod-network-cidr=10.250.0.0/16 --service-cidr=172.31.0.0/16 --apiserver-advertise-address 10.0.0.1 --apiserver-cert-extra-sans k8s.mydomain.com --apiserver-cert-extra-sans 192.168.1.13 --apiserver-cert-extra-sans 10.0.0.1 --ignore-preflight-errors=all
```

Next copy the kubectl config:

```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

And apply calico config

```
$ curl https://docs.projectcalico.org/v3.8/manifests/calico.yaml -O
$ vim calico.yaml

Change:
IP_AUTODETECTION_METHOD: "interface=wg*"
CALICO_IPV4POOL_CIDR: "10.250.0.0/16"
```

The `IP_AUTODETECTION_METHOD` needs to be wg0 so that all nodes will connect through the VPN, that will make sure it's working fine behind NAT, and all will be secure.

The `CALICO_IPV4POOL_CIDR` is needed to make sure all pods are created in the correct network.

When this is correct we can apply the network config:

```
$ kubectl apply -f calico.yaml
```

Keep note of the join command, so we can add another node.



# Adding another node

1) Prepare the node, in this example 'server2' with wireguard


VPN on server2:

```
[Interface]
PrivateKey = wPMsKBkqbdz1WBx8MhYM7/GwzYd6U7DWuef1FoeUdkg=
Address = 10.0.0.2/32

[Peer]
PublicKey = 8q+JKbrXDs86lnBvAl4lx6QiCzgoOOaAc7jtjz/lFBM=
AllowedIPs = 10.0.0.0/24
Endpoint = 123.123.123.123:55555
PersistentKeepalive = 15
```

Start the vpn, and make sure master and server2 can connect to each other:

```
master  > ping 10.0.0.2
server2 > ping 10.0.0.1
```

Now install `kubeadm` like before, and fix the `--node-ip` like we did before.

We can now join the node:

```
kubeadm join 10.0.0.1:6443 --token j9dg9i.u023uf023pr902u4 \
      --discovery-token-ca-cert-hash sha256:bb69ce798968473041754992927ce3b8154b526485055a0bf9fdd34c2aa34944
```

# Making sure everything is ok

Couple of things we can do to make sure it's all ok, first check all nodes are ready and using the vpn ip:

```
$ kubectl get node -o wide
root@k8s:~# kubectl get node -o wide
NAME      STATUS     ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
master    Ready      master   9h    v1.15.2   10.0.0.1      <none>        Ubuntu 18.04 LTS        4.15.18-13-pve               docker://19.3.1
server2   Ready      <none>   9h    v1.15.3   10.0.0.2      <none>        CentOS Linux 7 (Core)   3.10.0-957.27.2.el7.x86_64   docker://1.13.1
```

If the internal ip's are wrong, check your calico settings, beware that you might need to recreat the cluster using `kubeadm reset` first, as calico mentions changing the config later might not have effect.

Also make sure calico is fine:

```
$ curl -O -L  https://github.com/projectcalico/calicoctl/releases/download/v3.8.2/calicoctl
$ chmod +x calicoctl
$ DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config ./calicoctl get node -o wide
NAME      ASN       IPV4          IPV6
k8s       (64512)   10.0.0.1/32
server2   (64512)   10.0.0.2/32

$ DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config ./calicoctl node status
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 10.0.0.2     | node-to-node mesh | up    | 07:22:50 | Established |
+--------------+-------------------+-------+----------+-------------+
```

If it's not make sure calico settings are correct and firewall is ok:

```
master > $ nc -v 10.0.0.3 179
Connection to 10.0.0.3 179 port [tcp/bgp] succeeded!
```


After this you should have a k8s cluster running, from within proxmox.



References:
[1] https://medium.com/@kvaps/run-kubernetes-in-lxc-container-f04aa94b6c9c
[2] https://gist.github.com/kvaps/25f730e0ec39dd2e5749fb6b020e71fc
[3] https://stackoverflow.com/questions/55813994/install-and-create-a-kubernetes-cluster-on-lxc-proxmox
[4] https://github.com/corneliusweig/kubernetes-lxd
[5] https://www.wireguard.com/install/#ubuntu-module-tools
[6] https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
[7] https://blog.lbdg.me/proxmox-best-performance-disable-swappiness/
