# GKE+keycloak+nginx+lets-encrypt

**Goal: To setup keycloak with public free domain into GKE cluster using nginx controller instead of gce ingress controller.**

## Tools Used

- gcloud
- kustomize
- helm 3
- kubectl

## Create GKE cluster and storage class

```console
gcloud container clusters create keycloak-cluster --zone us-central1-c 
```

Configure `kubectl` with the newly created cluster

```console
gcloud container clusters get-credentials keycloak-cluser --zone us-central1-c --project your-project
```

To create storage class, need to depoloy nfs provisioner first.

Create filestore

```console
gcloud filestore instances create keycloak --file-share=name="keycloak",capacity=1T --network=name="default" --zone us-central1-c
```

Get the IP of the filestore

```console
gcloud filestore instances describe keycloak --zone us-central1-c --format='value(networks[0].ipAddresses[0])'
```

deploy nfs provisioner and create storage class name `nfs-client`

```console
DEMO_HOME=$(mktemp -d)

cat <<EOF >$DEMO_HOME/patch.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: not-important
data:
  nfsServer: FILESTORE_IP
  nfsPath: /keycloak
EOF


cat <<EOF >$DEMO_HOME/kustomization.yaml
resources:
- github.com/ffoysal/nfs-client-provisioner
patches:
- path: patch.yaml
  target:
    kind: ConfigMap
    name: nfs-provisioner-client-info
EOF


kustomize build $DEMO_HOME | kubectl apply -f -
```

## Deploy nginx-ingress controller

```console
# This stable helm repo is depreicated but still works with helm3
# Had to use it because it support same host name in multiple ingress resources

helm repo add stable https://charts.helm.sh/stable

helm install nginx-ingress stable/nginx-ingress --set rbac.create=true --set controller.publishService.enabled=true

# Get the external IP of the service nginx-ingress-controller
kubect get svc nginx-ingress-controller

# Promote the ephimeral IP to be static in GCE
gcloud compute addresses create nginx-ingress-lb --addresses <SVC-EXTERN AL_IP> --region us-central1

# Update the service to make the IP to be static
kubectl patch svc nginx-ingress-controller -p '{"spec": {"loadBalancerIP": "<SVC-EXTERN AL_IP>"}}'

```


## Create DOMAIN

The free domain can be found at https://freenom.com . Lets say domain name is `example.tk`

### Create Cloud DNS entry

Create a cloud DNS entry in GCE and give the domain name as `example.tk`

Take the NS (name server) entry from Cloud DNS and add them to freenom.com configuration

Add a recordset in google cloud DNS of type A with the load balancer IP and domain name `keycloak.example.tk`

## Deploy cert-manager for SSL

```console
kubectl create namespace cert-manager

helm repo add jetstack https://charts.jetstack.io

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.1.0 \
  --set installCRDs=true

# Create certificate issuer

cat <<EOF >$DEMO_HOME/issuer.yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: youremail@example.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

kubectl apply -f $DEMO_HOME/issuer.yaml
```

## Deploy keycloak

```console
cat <<EOF >$DEMO_HOME/values.yaml
ingress:
  enabled: true
  extraHosts: 
  - name: keycloak.example.tk
    path: /auth
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/issuer: "letsencrypt-prod"
  extraTls:
  - hosts:
    - keycloak.example.tk
    secretName: keycloak-tls
service:
  type: ClusterIP

proxyAddressForwarding: true

auth:
  adminUser: yourname
  adminPassword: changeme
global:
  storageClass: nfs-client
EOF

helm repo add bitnami https://charts.bitnami.com/bitnami

helm install keycloak bitnami/keycloak -f $DEMO_HOME/values.yaml

# nginx controller default buffer size is not enough for keycloak

cat <<EOF >$DEMO_HOME/cmap.yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-ingress-controller
data:
  proxy-buffer-size: "16k"
EOF

kubectl apply -f $DEMO_HOME/cmap.yaml

```

From browser access https://keycloak.example.tk/auth/