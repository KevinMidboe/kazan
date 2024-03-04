# Traefik Ingress Controller

Here is the values file for configuring traefik.

> [!IMPORTANT]  
> Requires configuring persistent storage, view `../nfs-storage`.

# Deployment

## 1. Add the Helm Repository and Update

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

## 2. Install Traefik with Helm

```bash
helm install traefik traefik/traefik -n traefik --values=values.yml
```

# Configuration


Configured to use cloudflare to resolve and generate certificates. This requires:
 - 1. cloudflare setup
 - 2. persistent storage
 - 3. 

### 1. Cloudflare ceritficate resolver



# Topology

Public --> Traefik on each kub host --> SVC or nginx or varnish



# SSL Termination

I want to have one box that is running nginx, one that is running varnish and ingress using traefik in kubernetes. How do I terminate a central place and route to correct host?


# Terminology

Ingress controller - a cluster wide controller for the Ingress resource.
Ingress resource - 



# Ingress Controllers

A ingress controller is required to enable Ingress Resources. Any number of ingress controllers using `ingress class` can be deployed to a cluster.

If you **do not** specify a `IngressClass` for a Ingrtess, and your cluster has exactly one IngressClass marked as default, then Kubernetes applies the clusters default IngressClass to the Ingress.
