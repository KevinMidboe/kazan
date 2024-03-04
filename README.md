# Kazan - Kubernetes cluster

On prem self-hosted kubernetes cluster running on [microk8s](https://microk8s.io/docs).

![cube gif](./assets/cube.gif)

## Table of Contents

- [Adding nodes](#adding-nodes)
- [Plugins](#plugins)
  - [MetalLB](#metallb)
  - [Helm](#helm)
- [Persistent Storage](#persistent-storage)
- [Ingress](#ingress)
  - [Configuration](#configuration)
  - [Deployment](#deployment)

# Adding nodes

## Provision and install microk8s
If this is first or additional node to be used for kubernetes we want to run base server configuration using playbook, replacing `KUBERNETES_HOST`:

```bash
ansible-playbook plays/base_server_setup.yml -i schleppe.ini -l KUBERNETES_HOST
```

Install microk8s using snap:

```bash
sudo snap install microk8s --classic --channel=1.27/stable
```

## Joining cluster

From a existing node run:

```bash
microk8s.add-node
```

Copy the output and join using command similar to the following on the new kubernetes node:

```bash
microk8s join kazan.schleppe:25000/60e891acfc38556ea569cd15d5e025a1/91a7ab376757
```

# Plugins

## MetalLB

MetallLB hooks into Kubernetes cluster and provides a network load-balancer implementation. 

```bash
microk8s.enable metallb:10.0.0.150-10.0.0.154
```

## Helm

Use helm charts to easier install managed systems and applications. 

```bash
microk8s.enable helm
```


# Persistent storage

To create a persistent volume claim (PVC) we need to first define a storage class. To not solve the issue of HA stoarge, a single NFS host is used. Setup expects a linux server service export path: `/srv/nfs`.

## Setup/Deployment

Define storage class pointing to NFS server:

```yaml
kubectl apply -f storage-nfs/storageClass.yml
```

> [!NOTE]
> PVC resources will reference `spec.storageClassName: nfs-csi`

# Ingress

Traefik is configured as default ingress, enabling dynamic configuration & SSL generation.


## Configuration

> [!IMPORTANT]
> Make sure that persistent storage is configured. Traefik is configured to expect `nfs-csi` storageClass already exists, follow steps above.

### Disable nginx ingress

Microk8s uses nginx as default ingress controller. We wish to replace this with traefik. Do disable plugin run:

```bash
microk8s.disable ingress
```

### Configure cloudflare certificate resolver

Add to `additionalArguments`:

```yaml
  - --certificatesresolvers.cloudflare.acme.dnschallenge.provider=cloudflare
  - --certificatesresolvers.cloudflare.acme.email=YOUR_EMAIL_HERE
  - --certificatesresolvers.cloudflare.acme.dnschallenge.resolvers=1.1.1.1
  - --certificatesresolvers.cloudflare.acme.storage=/ssl-certs/acme-cloudflare.json
```

It is important that persistent storage is configured and a `acme-cloudflare.json` file is manually touched and it's permissions updated. From the nfs server run:

```bash
cd /srv/nfs/PVC_VOLUME_NAME_HERE
sudo touch acme-cloudflare.json
sudo chown 65532:65532 acme-cloudflare.json
```

### Configure cloudflare credentials secret

Add to `env`:

```yaml
  - name: CF_API_EMAIL
    valueFrom:
      secretKeyRef:
        key: email
        name: cloudflare-credentials
  - name: CF_API_KEY
    valueFrom:
      secretKeyRef:
        key: apiKey
        name: cloudflare-credentials
```

Update following values from example secret file (`ingress-traefik/cloudflare-credentials.yml`):

```yaml
  email: YOUR_CLOUDFLARE_EMAIL
  apiKey: YOUR_CLOUDFLARE_API_KEY
```

Create a scoped api key for only only domain or use global api key.


Create secret object:

```bash
kubectl apply -f ingress-traefik/cloudflare-credentials.yml
```

### Configure persistent storage

Define persistent storage:

```yaml
persistence:
  enabled: true
  name: ssl-certs
  accessMode: ReadWriteOnce
  size: 1Gi
  storageClass: nfs-csi
  path: /ssl-certs
```

Define security context for all pods to not run as root, but as user & group `65532`. This solves NFS permissions in that we can ensure traefik has permissions to `acme-cloudflare.json` by setting file permissions to user & group: `65532`.

```yaml
securityContext:
  capabilities:
    drop: [ALL]
  readOnlyRootFilesystem: false
  runAsGroup: 65532
  runAsNonRoot: true
  runAsUser: 65532

podSecurityContext:
  fsGroup: 65532
```

## Deployment

### 1. Add the Helm Repository and Update

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

### 2. Install Traefik with Helm

```bash
helm install traefik traefik/traefik -n traefik --values=ingress-traefik/values.yml
```

