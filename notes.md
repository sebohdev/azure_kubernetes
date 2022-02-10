# Prerequisites

- Make sure to use Azure CLI
- Install kubectl
- Install helm
- k9s

# Connect to cluster

1. Configure Cluster

```bash
az aks get-credentials --resource-group Kubernetes --name hello_kubernetes
```

2. Validate

```bash
kubectl get nodes
```

# Deploy Application

1. Create files like in the 'manifests' directory
2. Apply folder to cluster

```bash
kubectl apply -f manifests/
```

3. Validate Deployement

```bash
kubectl get pods,services
```

# TLS Support

## Prep

No Need to create a new public IP Addres ... Sub Resource Group can be used!

1. Create a public ip address in the resource group

```bash
az network public-ip create -n NGINXAppIP -g MC_Kubernetes_hello_kubernetes_eastus2 --allocation-method Static --sku Standard
```

2. Give FQDN to Public IP

```bash
az network public-ip update -g MC_Kubernetes_hello_kubernetes_eastus2 -n NGINXAppIP --dns-name nginxrit --allocation-method Static
```

3. Verify DNS Setting

```bash
az network public-ip list -g MC_Kubernetes_hello_kubernetes_eastus2 --query "[?name=='NGINXAppIP'].[dnsSettings.fqdn]" -o tsv
```

4. Generate Key

```bash
openssl genrsa -out ca.key 2048
```

5. Create Certificate

```bash
openssl req -x509 -new -nodes -key ca.key -sha256 -subj "/CN=nginxrit.eastus2.cloudapp.azure.com" -days 1024 -out ca.crt -extensions v3_ca -config /etc/ssl/openssl.cnf
```

6. Create a secret

```bash
kubectl create secret tls nginx-tls --key=ca.key --cert=ca.crt
```

## Cert Manager

7. Add repo to help and update

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

8. Create ns (namespace) and install cert manager

```bash
kubectl create ns cert-manager

helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.3.1 --set installCRDs=true
```

9. Label the lets-encrypt namespace to disable resource validation

```bash
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
```

## Ingress

1. Add official repo to helm

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add stable https://charts.helm.sh/stable
helm repo update
```

2. Deploy ingress

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-basic --create-namespace --set controller.replicaCount=2 --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux --set controller.service.loadBalancerIP="20.122.107.21" --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"="nginxrit.eastus2.cloudapp.azure.com"
```

3. Check Status

```bash
 kubectl --namespace ingress-basic get services -o wide -w ingress-nginx-controller
```

## Cluster Issuer

1. Deploy TLS cert

```bash
kubectl apply -f tls/cluster-issuer.yml --namespace ingress-basic
```

## Update Services

1. Apply new manifest

```bash
kubectl apply -f tls/nginx-service.yml
```

## Apply Ingress for routing

```bash
kubectl apply -f tls/ingress.yml --namespace ingress-basic
```

## Test Certs

```bash
kubectl describe certificate tls-secret --namespace ingress-basic
```

all im gleichen namespace fuer ingress. Ansonsten geht es nicht