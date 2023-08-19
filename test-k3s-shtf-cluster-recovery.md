##### Related pages:
The official [K3s etcd restore](https://docs.k3s.io/cli/etcd-snapshot) whitepaper from [k3s.io](https://docs.k3s.io/)

---

During this trial we are going to restore a Kubernetes cluster in a complete Shit-Hit-The-Fan scenario by pretending that your on-premise datacenter has been nuked from orbit and now everything is gone. Therefore this is the utter-most hypothetical scenario and one I definitely wouldn't recommend. For best results if an etcd database restore is really needed, the `etcd` restore part with the `--cluster-reset` option should be done on the same master node as it is also explained bellow.

##### *"If you ever drop your keys into a river of molten lava, let 'em go, because man, they're gone."* <br> &nbsp;&nbsp;&nbsp;&nbsp;- Jack Handey, American humorist
<br>

However because we are all working remotely, you suddenly realize that you have a partial cluster backup on your laptop, because you've been poking around just a few days ago. That is, suddenly you realize that you have the most critical components to restore the `etcd` cluster database configuration and rebuild the cluster, that is, you have the `NODE-TOKEN` the `kube-config.yml` file and one `etcd` database snapshot.

Now let's begin with our little adventure...
<br><br>

### Original k3s cluster

---
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

##### What was on this cluster?
The original cluster was provisioned using the [Simple K3s cluster](simple-k3s-cluster.md) procedure, therefore this sets up the bare minimum for the new cluster. In particular since the old cluster used Longhorn module for storage, the new worker VMs should have at least one more disk prepared and formatted as LVM partitions and mounted in `/longhorn`. The rest of the cluster configuration is in the `etcd` database.

---
<br>

### Setup new VMs
The new VMs where you plan to restore everything. I just decided to keep the same names and just add the `-fail` suffix.

###### k3s-master-fail
```
k3s-master-fail.tomspirit.me       IN      A           172.16.0.60
k3s-fail.tomspirit.me - IN CNAME - k3s-master-fail.tomspirit.me
k3s-prometheus.tomspirit.me - IN CNAME - k3s-master-fail.tomspirit.me
k3s-alertmanager.tomspirit.me - IN CNAME - k3s-master-fail.tomspirit.me
k3s-grafana.tomspirit.me - IN CNAME - k3s-master-fail.tomspirit.me
```

###### k3s-worker01-fail
```
k3s-worker01-fail.tomspirit.me       IN      A           172.16.0.61
```

###### k3s-worker02-fail
```
k3s-worker02-fail.tomspirit.me       IN      A           172.16.0.63
```

###### k3s-worker03-fail
```
k3s-worker03-fail.tomspirit.me       IN      A           172.16.0.64
```
---


## General node preparation and important prerequisites
We'll begin with a standard kubernetes cluster preparation.<br><br>
On all nodes:
```bash
$ sudo ufw disable

# Used by the local-path-provisioner that comes with the cluster by default
$ sudo mkdir /local-path-provisioner

# Set the hostnames on all servers respectively
```bash
$ hostnamectl set-hostname k3s-master-fail.tomspirit.me
$ hostnamectl set-hostname k3s-worker01-fail.tomspirit.me
$ hostnamectl set-hostname k3s-worker02-fail.tomspirit.me
$ hostnamectl set-hostname k3s-worker03-fail.tomspirit.me
```

```bash
$ sudo apt update && sudo apt upgrade -y
$ sudo apt install -y open-iscsi nfs-common jq vim htop # Longhorn requirements and misc things
```
<br>

### The holy trinity
To get this done you will need the holy trinity of the k3s cluster (no I'm not religious):
1. The original `node-token` | Location is in: `/var/lib/rancher/k3s/server/node-token`
2. The original `kube-config.yml` | Original location in: `/etc/rancher/k3s/k3s.yaml` on the original master node.
3. Snapshot from the `etcd` database | Snapshot locations are on the master at: `/var/lib/rancher/k3s/server/db/snapshots`
<br>

### Setup the LVM partitions on all worker nodes
While I won't be covering this at this time, you can see how this is done at the: [Provision the worker nodes section](https://dev.azure.com/CloudWings/DevOps%20Adoption%20Framework/_wiki/wikis/DevOps-Adoption-Framework.wiki/140/Simple-K3s-cluster?anchor=provision-the-%60k3s%60-worker-nodes#) from the [Simple K3s cluster](/DevOps-Adoption-Framework/Solutions/Kubernetes-On%2DPremises-Scenarios/Simple-K3s-cluster) guidelines.

<br>


## Provision the `k3s` master
We will begin by installing a new cluster which will provision the k3s binaries and serve as a scaffolding where the old cluster will be restored:

```bash
$ curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.24.12+k3s1 sh -s - server \
--cluster-init \
--default-local-storage-path /local-path-provisioner \
--node-taint CriticalAddonsOnly=true:NoExecute \
--node-taint CriticalAddonsOnly=true:NoSchedule \
--tls-san 172.16.0.60 \
--tls-san k3s-fail.tomspirit.me \
--tls-san k3s-master.tomspirit.me \
--tls-san k3s-master-fail.tomspirit.me \
--disable traefik,metrics-server

# Give it 5 mins for the template cluster to fully provision then stop the k3s.service
$ systemctl stop k3s.service
```

Copy the original `node-token` from the master we are trying to restore **AND VERY IMPORTANT**, also make sure to add that same old token into the `K3S_TOKEN` environment variable:
```bash
cat .kube/node-token-k3s-original >/var/lib/rancher/k3s/server/node-token

## Example, your token will be different:
export K3S_TOKEN=K678bef6263dad0eedd6b449d28ca0cbbcacbfadb1c8bff2b07f0d3596468d8177c::server:51a0ab0422f7ac38ffc14b1001417252
```
<br>

## Reset the master using a snapshot from the other cluster
```bash
k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=/root/.kube/etcd-snapshots/etcd-snapshot-k3s-master.tomspirit.me-1692090004
```

Following a large output of messages and warnings hopefully the process should end with and IFNO message asking you to restart the `k3s.service`:
```bash
WARN[0000] remove /var/lib/rancher/k3s/agent/etc/k3s-agent-load-balancer.json: no such file or directory
WARN[0000] remove /var/lib/rancher/k3s/agent/etc/k3s-api-server-agent-load-balancer.json: no such file or directory
INFO[0000] Starting k3s v1.24.12+k3s1 (57e8adb5)
INFO[0000] Managed etcd cluster bootstrap already complete and initialized
INFO[0000] Starting temporary etcd to reconcile with datastore
{"level":"info","ts":"2023-08-16T15:36:46.872Z","caller":"embed/etcd.go:131","msg":"configuring peer listeners","listen-peer-urls":["http://127.0.0.1:2400"]}
{"level":"info","ts":"2023-08-16T15:36:46.873Z","caller":"embed/etcd.go:139","msg":"configuring client listeners","listen-client-urls":["http://127.0.0.1:2399"]}
# ...
# ...
# ... output omitted ...
# ...
# ...
INFO[0013] Managed etcd cluster membership has been reset, restart without --cluster-reset flag now. Backup and delete ${datadir}/server/db on each peer etcd server and rejoin the nodes
$


## Restart the services at the end
$ systemctl restart k3s.service
$ systemctl status k3s.service
● k3s.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2023-08-16 15:42:17 UTC; 48s ago
    ...
    ...
```

At this point we have a cluster with only one master (yeah that rymes well). If you've been doing the `etcd` restore on the old master node, at this time you are done. At most you will probably need to rejoin the old worker nodes and you can call it a day.

However, as we are restoring on a totaly new environment the old master node should be deleted from the configuration.
```bash
$ kubectl get nodes
NAME                              STATUS     ROLES                                          AGE   VERSION
k3s-master.tomspirit.me            NotReady   control-plane,etcd,master                      50d   v1.24.12+k3s1
k3s-master-fail.tomspirit.me       Ready      control-plane,etcd,master                      79s   v1.24.12+k3s1
k3s-worker01.tomspirit.me          NotReady   <none>                                         50d   v1.24.12+k3s1
k3s-worker02.tomspirit.me          NotReady   <none>                                         50d   v1.24.12+k3s1
k3s-worker03.tomspirit.me          NotReady   <none>                                         50d   v1.24.12+k3s1

$ kubectl delete node k3s-master.tomspirit.me
node "k3s-master.tomspirit.me" deleted

$ kubectl get nodes
NAME                              STATUS     ROLES                       AGE     VERSION
k3s-master-fail.tomspirit.me       Ready      control-plane,etcd,master   3m27s   v1.24.12+k3s1
k3s-worker01.tomspirit.me          NotReady   <none>                      50d     v1.24.12+k3s1
k3s-worker02.tomspirit.me          NotReady   <none>                      50d     v1.24.12+k3s1
k3s-worker03.tomspirit.me          NotReady   <none>                      50d     v1.24.12+k3s1
```
The old workers are understandably not available. We have other workers ready.
<br>

We also had `metallb` on the old environment which used tagged nodes to deploy the controler, therefore the tags have to be restored there too.
```bash
kubectl label node k3s-master-fail.tomspirit.me metallb-controller=true
```

You can add additional worker nodes to the cluster now
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.24.12+k3s1 \
K3S_URL=https://k3s-oject-fail.tomspirit.me:6443 \
K3S_TOKEN=K678bef6263dad0eedd6b449d28ca0cbbcacbfadb1c8bff2b07f0d3596468d8177c::server:51a0ab0422f7ac38ffc14b1001417252 \
sh -
```

After execuing the join command check the status of the cluster:
```bash
$ kubectl get nodes
NAME                              STATUS     ROLES                                          AGE   VERSION
k3s-master-fail.tomspirit.me       Ready      control-plane,etcd,master                      14m   v1.24.12+k3s1
k3s-worker01.tomspirit.me          NotReady   <none>                                         50d   v1.24.12+k3s1
k3s-worker02.tomspirit.me          NotReady   <none>                                         50d   v1.24.12+k3s1
k3s-worker03.tomspirit.me          NotReady   <none>                                         50d   v1.24.12+k3s1

## Shortly after the new nodes will appear
$ kubectl get nodes
NAME                              STATUS     ROLES                                          AGE   VERSION
k3s-master-fail.tomspirit.me       Ready      control-plane,etcd,master                      15m   v1.24.12+k3s1
k3s-worker01.tomspirit.me          NotReady   <none>                                         50d   v1.24.12+k3s1
k3s-worker01-fail.tomspirit.me     Ready      <none>                                         12s   v1.24.12+k3s1
k3s-worker02.tomspirit.me          NotReady   <none>                                         50d   v1.24.12+k3s1
k3s-worker02-fail.tomspirit.me     Ready      <none>                                         10s   v1.24.12+k3s1
k3s-worker03.tomspirit.me          NotReady   <none>                                         50d   v1.24.12+k3s1
k3s-worker03-fail.tomspirit.me     Ready      <none>                                         12s   v1.24.12+k3s1
```

Delete the old nodes:
```bash
$ kubectl get nodes
NAME                              STATUS     ROLES                                          AGE   VERSION
k3s-master-fail.tomspirit.me       Ready      control-plane,etcd,master                      41m   v1.24.12+k3s1
k3s-worker01.tomspirit.me          NotReady   <none>                                         50d   v1.24.12+k3s1
k3s-worker01-fail.tomspirit.me     Ready      <none>                                         26m   v1.24.12+k3s1
k3s-worker02.tomspirit.me          NotReady   <none>                                         50d   v1.24.12+k3s1
k3s-worker02-fail.tomspirit.me     Ready      <none>                                         26m   v1.24.12+k3s1
k3s-worker03.tomspirit.me          NotReady   <none>                                         50d   v1.24.12+k3s1
k3s-worker03-fail.tomspirit.me     Ready      <none>                                         26m   v1.24.12+k3s1

$ kubectl delete node k3s-worker01.tomspirit.me
node "k3s-worker01.tomspirit.me" deleted

$ kubectl delete node k3s-worker02.tomspirit.me
node "k3s-worker02.tomspirit.me" deleted

$ kubectl delete node k3s-worker03.tomspirit.me
node "k3s-worker03.tomspirit.me" deleted

$ kubectl get nodes
NAME                              STATUS   ROLES                                          AGE   VERSION
k3s-master-fail.tomspirit.me       Ready    control-plane,etcd,master   42m   v1.24.12+k3s1
k3s-worker01-fail.tomspirit.me     Ready    <none>                                         27m   v1.24.12+k3s1
k3s-worker02-fail.tomspirit.me     Ready    <none>                                         27m   v1.24.12+k3s1
k3s-worker03-fail.tomspirit.me     Ready    <none>                                         27m   v1.24.12+k3s1
```
<br>


## Aftermath consequences
As you remove and add the old nodes the cluster will continue to provision all of the deployments and services it had before. You'll need to give it some time for it to stabilize. Bellow are some things that I noticed I had to still fix manually:

### Prometheus stack
I had to redeploy this module (which are three modules actually) using the helm chart and update the values. Reason being, the nodeExporter and the prometheus-node-exporter in the values file had tolerations: for the master node. For some reason when the `kube-prometheus-stack` was deployed, these tolerations didn't carry over the restored version. I'm talking about this section of the [`values-kube-prometheus-stack.yml`](/DevOps-Adoption-Framework/Solutions/Kubernetes-On%2DPremises-Scenarios/Simple-K3s-cluster/values%2Dkube%2Dprometheus%2Dstack.yml) file:

```yaml
## Deploy node exporter as a daemonset to all nodes
##
nodeExporter:
  enabled: true

  tolerations:
  - key: CriticalAddonsOnly
    operator: Exists
    effect: NoExecute
  - key: CriticalAddonsOnly
    operator: Exists
    effect: NoSchedule

## Configuration for prometheus-node-exporter subchart
##
prometheus-node-exporter:
  namespaceOverride: ""

  tolerations:
  - key: CriticalAddonsOnly
    operator: Exists
    effect: NoExecute
  - key: CriticalAddonsOnly
    operator: Exists
    effect: NoSchedule
```

### Longhorn
This is a big one. If you didn't have any backup from anything on the Longhorn storage, you dropped your keys into the molten lava and that data it is gone, it's null and void. Thing to note is to always have an external backup available elsewhere, even if your deployments are on the cloud, always export your backup to another cloud region. After all, no one guaranties that any AWS datacenter will remain intact forever.

### MetalLB
This is another point where manual interaction might be needed. If the new VMs are in another network/region, the MetalLB `IPAddressPool` should be updated with the new IP Ranges accordingly.

### Development and Deployment Pipelines
Depending on the previous setup these will have to be recreated (hopefully not from scratch) to match the new cluster location.

