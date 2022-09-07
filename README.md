# Setup Kafka on Kubernetes using NFS and External Access using NodePort

## Setup NFS Server for PV

`sudo apt-get install nfs-kernel-server`

```
sudo mkdir -p /srv/nfs
sudo chown nobody:nogroup /srv/nfs
sudo chmod 0777 /srv/nfs
```

Edit `/etc/exports` file, making sure that the IP addresses of all your MicroK8s nodes are able to mount this share

```
sudo mv /etc/exports /etc/exports.bak
echo '/srv/nfs *(rw,sync,no_subtree_check)' | sudo tee /etc/exports
sudo exportfs -rav
```

Restart nfs server
`sudo systemctl restart nfs-kernel-server`

## Install CSI driver for NFS

```
microk8s helm3 repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
microk8s helm3 repo update
microk8s helm3 install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
    --namespace kube-system \
    --set kubeletDir=/var/snap/microk8s/common/var/lib/kubelet
```

Wait for csi controller pod to startup.
`microk8s kubectl wait pod --selector app.kubernetes.io/name=csi-driver-nfs --for condition=ready --namespace kube-system`

List available csi drivers.
`microk8s kubectl get csidrivers`

```
NAME             ATTACHREQUIRED   PODINFOONMOUNT   STORAGECAPACITY   TOKENREQUESTS   REQUIRESREPUBLISH   MODES        AGE
nfs.csi.k8s.io   false            false            false             <unset>         false               Persistent   66s
```

## Create a StorageClass for NFS

Edit `nfs-csi.yaml`

```
mkdir /app/nfs
cd /app/nfs
nano nfs-csi.yaml
```

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.1.30
  share: /srv/nfs
reclaimPolicy: Retain
volumeBindingMode: Immediate
mountOptions:
  - hard
  - nfsvers=4.1
```

Apply on kubernetes
`microk8s kubectl apply -f - < nfs-csi.yaml`

## Create PV for kafka

```
sudo mkdir -p /srv/nfs/app-infra/kafka
sudo chown nobody:nogroup /srv/nfs/app-infra/kafka
sudo chmod 0777 /srv/nfs/app-infra/kafka
```

```
cd /app/app-infra
mkdir kafka
cd kafka
```

Create `app-infra-kafka.yaml`
`nano app-infra-kafka.yaml`

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-infra-kafka
spec:
  capacity:
    storage: 8Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-csi
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /srv/nfs/app-infra/kafka
    server: 192.168.1.30
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-infra-kafka
spec:
  storageClassName: nfs-csi
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 8Gi
  volumeName: app-infra-kafka
```

Apply
`microk8s kubectl -n app-infra apply -f - < app-infra-kafka.yaml`

## Create PV for kafka-zookeeper

```
sudo mkdir -p /srv/nfs/app-infra/kafka-zookeeper
sudo chown nobody:nogroup /srv/nfs/app-infra/kafka-zookeeper
sudo chmod 0777 /srv/nfs/app-infra/kafka-zookeeper
```

```
cd /app/app-infra
mkdir kafka
cd kafka
```

Create `app-infra-kafka-zookeeper.yaml`
`nano app-infra-kafka-zookeeper.yaml`

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-infra-kafka-zookeeper
spec:
  capacity:
    storage: 8Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-csi
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /srv/nfs/app-infra/kafka-zookeeper
    server: 192.168.1.30
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-infra-kafka-zookeeper
spec:
  storageClassName: nfs-csi
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 8Gi
  volumeName: app-infra-kafka-zookeeper
```

Apply
`microk8s kubectl -n app-infra apply -f - < app-infra-kafka-zookeeper.yaml`

## Configure External Access using NodePort

Create `values.yaml`

```
zookeeper:
  service:
    type: NodePort
    nodePorts:
      client: 30011
externalAccess:
  enabled: true
  service:
    type: NodePort
    nodePorts:
      - 30010
    autoDiscovery:
      enabled: true
    useHostIPs: true
    # Use localhost for Kubernetes on Docker Desktop
    # domain: localhost
persistence:
  enabled: true
  existingClaim: app-infra-kafka
zookeeper:
  service:
    type: NodePort
    nodePorts:
      client: 30011
  persistence:
    existingClaim: app-infra-kafka-zookeeper
```

## Install

Add bitnami repository

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm -n app-infra upgrade --install --create-namespace kafka bitnami/kafka -f values.yaml
```

## Uninstall

`helm -n app-infra uninstall kafka`
