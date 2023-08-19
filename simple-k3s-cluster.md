#### Related articles
The links bellow are to be used in conjunction with this guide:

- [Test-Case - MetalLB](test-metallb.md)
- [Test-Case - Longhorn Storage](test-longhorn.md)
- [Test-Case - `cert-manager` and `ingress-nginx`](test-cert-manager-self-signed-pki.md)
- [K3s cluster recovery](test-k3s-shtf-cluster-recovery.md)
---

<br>

## Preface
The following guide is a simple yet effective approach to build an on-premises Kubernetes cluster using the SUSE Rancher's K3s Kubernetes engine. It is a simple yet effective solution for many scenarios and it will work great even for on-premises production.

This guide takes into consideration the following scenario.<br>

1. **Only one master node.**<br>
Considering the fact that most on-premise datacenters use some virtualization solution like VMware vSphere, KVM with Proxmox etc, all of those already have some backup in place. Therefore the VMs will be backed up already. <br><br>
2. **At least three worker nodes**<br>
Three are the recommended minimum for the Kubernetes storage module - Longhorn.

Finally don't forget to visit the [Official Documentation](https://docs.k3s.io/) for more configuration or architectural design options
<br><br>

### Minimum Recommended Requirements
The requirements greatly depend on the project in question and the application performance expectations. This cluster has even been tested to be working in a home lab on a stronger desktop computer with a VirtualBox.

However for small to medium teams working on a small start-up project I could only assume the following would be the bare minimum:
<br><br>

#### VM Requirements
**Operating System:** This has been tested on Ubuntu22.04

For more serious setup each VM in use should have the bellow bare minimums:
1. CPU - 4x Virtual
2. RAM - 8GB per VM (probably 16GB for the master node)

**Storage**
1. Disk 40GB for the system OS volume
2. Disk 100GB or more provisioned as LVM volume and mounted in `/longhorn`

**Network**
1. Network speed - Project specific
2. A reserved network pool of Routable IP address for the MetalLB where the DHCP will not assign any address. In this example the VMs of the cluster are in the `172.16.0.0/24` network. I decided to take a chunk of IPs by dividing it into four subnets and I will be using the last subnet `/26` subnet for the MetalLB i.e `172.16.0.192/26`. This gives me 62 usable IP addresses for use by MetalLB for any future deployments.

#### Underlying hardware/fabric requirements
As this is planned for on-premise scenarios it is highly recommended for the underlying hypervisor to have snapshot capabilities and an external backup for each VM respectively. 
<br><br>

#### VERY IMPORTANT 
---

- It is best that all VMs in the cluster are backed up externally using a Hypervisor VM backup solution. Example would be Veeam Backup or VMware vSphere clusters.
- The master node VM must always have daily backups. This is where the ETCD database which contains the cluster configuration is located. The ETCD service on the master nodes does it's own daily snapshots at `/var/lib/rancher/k3s/server/db/snapshots/`
- The worker nodes does not have to be backed up, however if the application in question uses the longhorn storage, this should have its own backup (application specific). The Longhorn storage module itself has backup options which are out of the scope of this guide.

---
* **Finally, once you provision the K3s master node the node token (`K3S_TOKEN`), the etcd db snapshots mentioned above, and the kube config `k3s.yaml` file are to be considered the holly trinity of the kubernetes cluster and should be securely backed up, and kept at a save place, until you no longer need the cluster.**
##### Lose these and you lose the cluster. You've been warned!
---
<br>

### Node details
All nodes should have their IP and DNS hostnames configured in the local DNS for proper name resolution:

###### k3s-master
```
k3s-master.tomspirit.me                     IN      A           172.16.0.50
k3s.tomspirit.me                            IN      CNAME       k3s-master.tomspirit.me  ## This is the k3s cluster URL
k3s-prometheus.tomspirit.me             IN      CNAME       k3s-master.tomspirit.me
k3s-alertmanager.tomspirit.me           IN      CNAME       k3s-master.tomspirit.me
k3s-grafana.tomspirit.me                IN      CNAME       k3s-master.tomspirit.me
```

###### k3s-worker01
```
k3s-worker01.tomspirit.me     IN      A           172.16.0.51
```
###### k3s-worker02
```
k3s-worker02.tomspirit.me     IN      A           172.16.0.52
```

###### k3s-worker03
```
k3s-worker03.tomspirit.me     IN      A           172.16.0.53
```
---
<br>

## General node preparation
This should be done on all nodes:
```bash
$ sudo ufw disable

# Used by the local-path-provisioner that comes with the cluster by default
$ sudo mkdir /local-path-provisioner

# Set the hostnames on all servers respectively
$ sudo hostnamectl set-hostname SERVER-FQDN

$ sudo apt update && sudo apt upgrade -y
$ sudo apt install -y open-iscsi nfs-common jq vim htop # Longhorn requirements and misc things
```
<br>

## Provision the `k3s` master
By default `k3s` deployes `traefic` and `metrics-server` as part of the deployment. These will be disabled because `nginx-ingress` and `metallb` will be used instead.

```bash
$ curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.24.12+k3s1 sh -s - server \
--cluster-init \
--default-local-storage-path /local-path-provisioner \
--node-taint CriticalAddonsOnly=true:NoExecute \
--node-taint CriticalAddonsOnly=true:NoSchedule \
--tls-san 172.16.0.50 \
--tls-san k3s.tomspirit.me \
--tls-san k3s-master.tomspirit.me \
--disable traefik,metrics-server

# Get the node-token and set the kubeconfig to be readable for everyone
$ sudo cat /var/lib/rancher/k3s/server/node-token 
## Take note of the node token

$ sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

Get the node-token which you will need for adding the worker nodes at the [Add worker nodes](#add-worker-nodes) section bellow:
```bash
# Get the node-token and set the kubeconfig to be readable for everyone
$ sudo cat /var/lib/rancher/k3s/server/node-token  ## Take note of the node token and use it at the $K3S_TOKEN var bellow

$ sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```
<br>

## Provision the `k3s` worker nodes
The worker nodes will need some additional preparation mostly for the storage part.

### K3s storage preparation
The cluster by default uses the so-called `local-path-provisioner` with its location at `/local-path-provisioner`. This is a good storage for testing. For more serious use-cases there will be a [Longhorn](#longhorn-storage-install) kubernetes storage module installed bellow. 

For this reason additional storage partitions should be prepared as `lvm ext4` and mounted on `/longhorn`. The VMs in use for this had an additional virtual disk shown as `/dev/sdb` on each VM.
In short, the additional storage partition has been provisioned using the following commands:
```bash
## Create the PV (Physical Volume)
$ pvcreate /dev/sdb && pvdisplay /dev/sdb

## Create the VG (Volume Group)
$ vgcreate longhorn-vg /dev/sdb && vgdisplay longhorn-vg

## Create he LV (Logical Volume)
##   The `--extents 38399`` value has been obtained from the PE value from the `vgdisplay` command
$ lvcreate --extents 38399 -n longhorn-lv longhorn-vg && lvdisplay /dev/longhorn-vg/longhorn-lv

## Format and mount the lvm partition as ext4 file system
$ mkfs.ext4 /dev/longhorn-vg/longhorn-lv
$ mkdir /longhorn && mount /dev/longhorn-vg/longhorn-lv /longhorn && df -Th

## Update the /etc/fstab file
$ cat /etc/fstab >fstab.20230809.bak && echo '/dev/mapper/longhorn--vg-longhorn--lv /longhorn ext4 defaults 0 0' | tee -a /etc/fstab && echo -n "\n\n" && cat /etc/fstab

## REBOOT the system at the end to make sure everythingn is working normally and
##  confirm the partition is mounted using `df -h` or `lsblk` commands
$ systemctl reboot
$ lsblk
#...
#... output omited ...
#...
sdb                           8:16   0   150G  0 disk
└─longhorn--vg-longhorn--lv 253:0    0   150G  0 lvm  /longhorn
```

### Add worker nodes to the cluster
Execute the bellow command on each of the worker nodes, but make sure to update the `K3S_TOKEN` variable which can be obtained from the master node at the previous step:
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.24.12+k3s1 \
K3S_URL=https://k3s.tomspirit.me:6443 \
K3S_TOKEN=VERY_LARGE_NODE_TOKEN_HERE \
sh -
```
<br>

## Verify the cluster
The bellow commands should return positive status, a cluster with 4 nodes in total:
```bash
$ kubectl get nodes
$ kubectl get pods --all-namespaces
$ kubectl cluster-info
```
---
<br>

## Install additional packages
Additional packages are important for the functionality of the whole cluster. This cluster will feature the following modules added:
1. [`cert-manager`](#cert-manager) - Provide SSL/TLS Certificate functionality for other services;
2. [`ingress-nginx`](#ingress-nginx) - Provides L7 Ingress load balancing;
3. [`metallb`](#metallb) - provides L4 load balancing;
4. [`prometheus-monitoring`](#prometheus-monitoring) - Adds Prometheus, Grafana and Alert Manager monitoring components
5. [`Longhorn`](#longhorn-storage-install) - Adds Kubernetes storage capabilities
<br>
<br>


### `cert-manager`
For more info visit:
- `cert-manager` [documentation](https://cert-manager.io/docs/)
- `cert-manager` [helm repo](https://artifacthub.io/packages/helm/cert-manager/cert-manager)
```bash
$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml

$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update

$ helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.12.0
```

Once the `cert-manager` is installed create a default self signed ClusterIssuer for the whole cluster. This gives the capability to issue self signed certificates a simple TLS capability:
```bash
$ cat >cert-manager-ClusterIssuer.yml <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
EOF

$ kubectl apply -f cert-manager-ClusterIssuer.yml
```
<br>

### `ingress-nginx`
For more info visit:
- `ingress-nginx` [helm repo](https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx)

```bash
# nginx ingress 
# https://kubernetes.github.io/ingress-nginx/deploy/
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx 
$ helm repo update
$ helm search repo ingress-nginx --versions

$ cat >values-ingress-nginx.yml <<EOF
controller:
  ingressClass: nginx
  ingressClassResource:
    default: 'true'
  
  service:
    type: LoadBalancer

  admissionWebhooks:
    certManager:
      enable: 'true'

  metrics:
    enabled: true
    prometheusRule:
      enable: 'true'

  config:
    allow-snippet-annotations: 'true'
    ssl-redirect: 'false'
    hsts: 'false'
EOF

$ helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --version 4.7.1 \
  --values values-ingress-nginx.yml
```

**NOTE:**<br>
The `cert-manager` and `ingress-nginx` test at the additional section bellow is actually done automatically done during the [prometheus-monitoring](#prometheus-monitoring) section, as it creates a self-signed certificates for the prometheus stack and implements them at the `ingress-nginx`. It can still be used however to test an internal PKI implementation for a separate namespace.

---

##### Additional section: 
[Test ingress-nginx and cert-manager self signed certificates](test-cert-manager-self-signed-pki.md)

---
<br>

### Prometheus monitoring
For more info visit:
- `prometheus-community` [helm repo](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack)
- `prometheus-community` [github repo](https://github.com/prometheus-community/helm-charts/tree/kube-prometheus-stack-16.0.1/charts/kube-prometheus-stack)

As per my experience this is the most boring and complex part of the setup, mainly because the prometheus stack is actuall 3 or more products (based on your settings) bundled into one helm chart.

First create the monitoring namespace and the TLS Certificate which will be used by the prometheus stack:
```bash
$ cat >monitoring-namespace.yml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
EOF

$ kubectl apply -f monitoring-namespace.yml
```

```bash
$ cat >prometheus-stack-cert.yml <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: prometheus-stack-certificate
  namespace: monitoring
spec:
  secretName: prometheus-stack-certificate-rsa-secret
  
  duration: 87600h #10y
  renewBefore: 3600h

  subject:
    organizations:
      - tomspirit.me
  
  isCA: False

  privateKey:
    #algorithm: ECDSA
    #size: 256

    algorithm: RSA
    encoding: PKCS1
    size: 2048

  usages:
    - server auth
    - client auth

  dnsNames:
    - k3s-prometheus.tomspirit.me
    - k3s-alertmanager.tomspirit.me
    - k3s-grafana.tomspirit.me

  issuerRef:
    name: selfsigned-cluster-issuer
    kind: ClusterIssuer
    group: cert-manager.io
EOF

$ kubectl apply -f prometheus-stack-cert.yml
```

To install the `prometheus-stack` you will need to properly configure the `.values` file. If you are to lazy to do that yourself I can understand that compleately, and to counter that I also have this whole project on my [GitHub profile](https://github.com/spirit986/k3s-cluster) so you can take the [`values-kube-prometheus-stack.yml`](https://github.com/spirit986/k3s-cluster/blob/main/values-kube-prometheus-stack.yml) from there.

Install the `prometheus-stack` using `helm`:
```bash
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts 
$ helm repo update 

$ helm install kube-prometheus-stack \
  -f values-kube-prometheus-stack.yml \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version 46.8.0 
```

To test the implementation once all of the pods have been successfully deployed browse to: `https://k3s-grafana.tomspirit.me` and login with the admin username and password used from the `adminPassword` value from the `values-kube-prometheus-stack.yml` file.
<br>
<br>

### Longhorn storage install
For more info visit:
- [Longhorn docs](https://longhorn.io/docs/1.4.2/deploy/install/)
```bash
$ curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.4.1/scripts/environment_check.sh | bash 

$ helm repo add longhorn https://charts.longhorn.io 
$ helm repo update
$ helm install longhorn longhorn/longhorn \
  --namespace longhorn-system --create-namespace \
  --version 1.4.2 
```
<br>

#### Longhorn Configuration
The configuration includes adjusting some settings from the UI as well as configuring additional storage classes to accommodate particular usage scenarios.

The default class `longhorn` can be used too, but it shouldn't be altered, as per their documentation.
<br>

#### Accessing the Longhorn UI
To access the longhorn UI it is best if you use Lens and create a port forward on the longhorn frontend service.

* From the Lens UI select `Network` -> `Services`.
* From the top right `Namespace` drop down menu select `longhorn-system`.
* From the list of services locate the `longhorn-frontend` service and click on it.
* On the right pannel that opens at the **Connection** section there is the **Ports** sub-section indicating `80:http/TCP`. Click on the **Forward** button to enable port forward to the interface.
* Once you are finished working, click **Stop/Remove** on the same button, or delete the port forwarding object from the `Network` -> `Port Forwarding` section.
<br>

#### Initial configuration
From the Longhorn UI the system has been configured with `/longhorn` as its main storage device. This location is mounted on a separate LVM volume from within the system.

The default storage device `/var/lib/longhorn` has been disabled as this resides on the `/(root)` location of the system.

For each of the nodes at the **Node** section of the UI the following disk configuration has been set:
- Default disk (usually named) `default-disk-fd0100000`, with path `/var/lib/longhorn`, set scheduling to `DISABLED`, added label `do-not-use`, for future reference.
- Added the additional LVM partition which was prepared earlier as Longhorn `disk-1`,with path `/longhorn`. Scheduling set to `ENABLED`, added label `LVM` and 25G storage reserved.
<br>

##### Best practices
The following is just a section from the [Longhorn Best Practices](https://longhorn.io/docs/1.4.2/best-practices) from their official documentation.

* Replica Node Level Soft Anti-Affinity: `FALSE`
* Allow Volume Creation with Degraded Availability: `FALSE`
<br>

#### Creating additional storage classes
More information of the settings applied can be found at:
* [Data Locality](https://longhorn.io/docs/1.4.2/references/settings/#default-data-locality)
* [Reclaim Policy](https://longhorn.io/docs/1.4.2/volumes-and-nodes/delete-volumes/#deleting-volumes-through-kubernetes)

```bash
$ cat >longhorn-storage-classes.yml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-best-effort-reclaim-delete
  annotations:
    storageclass.kubernetes.io/is-default-class: 'false'

provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate

parameters:
  dataLocality: best-effort
  fromBackup: ''
  fsType: ext4
  numberOfReplicas: '3'
  staleReplicaTimeout: '2880'

---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-best-effort-reclaim-retain
  annotations:
    storageclass.kubernetes.io/is-default-class: 'false'

provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: Immediate

parameters:
  dataLocality: best-effort
  fromBackup: ''
  fsType: ext4
  numberOfReplicas: '3'
  staleReplicaTimeout: '2880'
EOF

$ kubectl apply -f longhorn-storage-classes.yml
```

---

##### Additional section: 
[Test Longhorn volume provisioning](test-longhorn.md)

---

<br>

### MetalLB
For more info visit:
- [Installation reference](https://metallb.universe.tf/installation/#installation-with-helm)
<br>

Label the node and generate the `metallb` values file:
```bash
$ kubectl label node k3s-master.tomspirit.me metallb-controller=true

$ cat >values-metallb.yml <<EOF
loadBalancerClass: "metallb"

controller:
  nodeSelector:
    metallb-controller: "true"

  tolerations:
    - key: CriticalAddonsOnly
      operator: Exists
      effect: NoExecute
    - key: CriticalAddonsOnly
      operator: Exists
      effect: NoSchedule

speaker:
  frr:
    enabled: false
EOF
```
<br>

Deploy metallb using their helm chart:
```bash
$ helm repo add metallb https://metallb.github.io/metallb
$ helm repo update
$ helm install metallb metallb/metallb --namespace metallb-system --create-namespace -f values-metallb.yml --version 0.13.10
```
<br>

Configure the IP address pool and the L2 advertisement | [Reference](https://metallb.universe.tf/configuration/#layer-2-configuration)
```bash
$ cat >IPAddressPool.yml <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: dev-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.16.0.192/26

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2-advertisement
  namespace: metallb-system
EOF

$ kubectl apply -f IPAddressPool.yml
```

---

##### Additional section: 

[Test MetalLB deployment](test-metallb.md)

---
