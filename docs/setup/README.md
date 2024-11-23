# Cluster setup

## Pre-requisites

Before deploying the cluster and deploying apps you need to install few things on your host:

- [k3sup](https://github.com/alexellis/k3sup)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- [cilium](https://cilium.io/)

### Install k3sup

```bash
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/

k3sup --help
```

### Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### Install cilium

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

## Prepare nodes

```bash
sudo cp /home/user/.ssh/authorized_keys /root/.ssh/authorized_keys
sudo echo "elevator=deadline cgroup_memory=1 cgroup_enable=memory" >> /boot/firmware/cmdline.txt # Only on raspberry pi
apt install jq curl open-iscsi -y
reboot
```
## Deploying HA k3s cluster

On your host:

```bash
k3sup install --sudo=false \
    --ip 10.1.0.1 \
    --tls-san=10.1.0.4 \
    --cluster \
    --k3s-extra-args '--flannel-iface=eth0 --disable=traefik --disable=servicelb --disable=local-storage --flannel-backend=none --disable-network-policy --node-ip=10.1.0.1 --cluster-cidr=10.42.0.0/16 --service-cidr=10.43.0.0/16' \
    --print-command

mkdir -p ~/.kube/
cat kubeconfig > ~/.kube/config
```

On the 1st Node (10.1.0.1):

```bash
curl https://kube-vip.io/manifests/rbac.yaml > /var/lib/rancher/k3s/server/manifests/kube-vip-rbac.yaml
export VIP=10.1.0.4
export INTERFACE=eth0
KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
alias kube-vip="ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"
kube-vip manifest daemonset \
    --interface $INTERFACE \
    --address $VIP \
    --inCluster \
    --taint \
    --controlplane \
    --services \
    --arp \
    --leaderElection > /var/lib/rancher/k3s/server/manifests/kube-vip-ds.yaml
```

On your host:

```bash
k3sup join --sudo=false \
    --ip 10.1.0.2 \
    --server \
    --server-ip 10.1.0.4 \
    --k3s-extra-args '--flannel-iface=eth0 --disable=traefik --disable=servicelb --disable=local-storage --flannel-backend=none --disable-network-policy --node-ip=10.1.0.2 --cluster-cidr=10.42.0.0/16 --service-cidr=10.43.0.0/16' \
    --print-command

k3sup join --sudo=false \
    --ip 10.1.0.3 \
    --server \
    --server-ip 10.1.0.4 \
    --k3s-extra-args '--flannel-iface=eth0 --disable=traefik --disable=servicelb --disable=local-storage --flannel-backend=none --disable-network-policy --node-ip=10.1.0.3 --cluster-cidr=10.42.0.0/16 --service-cidr=10.43.0.0/16' \
    --print-command
```

## Setup cilium

Install cilium:

```bash
cilium install --helm-set ipam.operator.clusterPoolIPv4PodCIDRList="10.42.0.0/16" --helm-set ipam.operator.clusterPoolIPv4MaskSize=24 --helm-set envoy.enabled=false
```

Setup BGP:

```bash
cilium config set enable-bgp-control-plane true
sleep 20
pod=$(kubectl get pods -n kube-system -o name | grep cilium-operator)
kubectl delete -n kube-system $pod
sleep 60
kubectl apply -f setup/cilium/peering-policy.yml
sleep 10
kubectl label nodes $(kubectl get nodes -o=custom-columns=NAME:.metadata.name --no-headers) bgp-policy=default
kubectl apply -f setup/cilium/ip-pool.yml
```

# Setup fluxcd

Once everything is up:

```bash
flux bootstrap github --owner=phorge-fr --repository=core-k8s --branch=main --path=cluster/core
```