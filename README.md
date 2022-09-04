# Setup Kafka on Kubernetes

## Create configuration for external access

Create `values.yaml`

```
mkdir kafka
cd kafka
nano values.yaml
```

Paste into `values.yaml`

```
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
zookeeper:
  service:
    type: NodePort
    nodePorts:
      client: 30011
```

## Install

Add bitnami repository
`helm repo add bitnami https://charts.bitnami.com/bitnami`

`helm repo update`

`helm -n ktcmaai-infra upgrade --install --create-namespace kafka bitnami/kafka -f values.yaml`
