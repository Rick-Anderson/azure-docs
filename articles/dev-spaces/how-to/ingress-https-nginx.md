---
title: "Use a custom NGINX ingress controller and configure HTTPS"
services: azure-dev-spaces
ms.date: "12/10/2019"
ms.topic: "conceptual"
description: "Learn how to configure Azure Dev Spaces to use a custom NGINX ingress controller and configure HTTPS using that ingress controller"
keywords: "Docker, Kubernetes, Azure, AKS, Azure Kubernetes Service, containers, Helm, service mesh, service mesh routing, kubectl, k8s"
ms.custom: devx-track-js, devx-track-azurecli
---

# Use a custom NGINX ingress controller and configure HTTPS

[!INCLUDE [Azure Dev Spaces deprecation](../../../includes/dev-spaces-deprecation.md)]

This article shows you how to configure Azure Dev Spaces to use a custom NGINX ingress controller. This article also shows you how to configure that custom ingress controller to use HTTPS.

## Prerequisites

* An Azure subscription. If you don't have one, you can create a [free account][azure-account-create].
* [Azure CLI installed][az-cli].
* Azure Kubernetes Service (AKS) cluster with Azure Dev Spaces enabled.
* [kubectl][kubectl] installed.
* [Helm 3 installed][helm-installed].
* [A custom domain][custom-domain] with a [DNS Zone][dns-zone].  This article assumes the custom domain and DNS Zone are in the same resource group as your AKS cluster, but it is possible to use a custom domain and DNS Zone in a different resource group.

## Configure a custom NGINX ingress controller

Connect to your cluster using [kubectl][kubectl], the Kubernetes command-line client. To configure `kubectl` to connect to your Kubernetes cluster, use the [az aks get-credentials][az-aks-get-credentials] command. This command downloads credentials and configures the Kubernetes CLI to use them.

```azurecli
az aks get-credentials --resource-group myResourceGroup --name myAKS
```

To verify the connection to your cluster, use the [kubectl get][kubectl-get] command to return a list of the cluster nodes.

```console
kubectl get nodes
NAME                                STATUS   ROLES   AGE    VERSION
aks-nodepool1-12345678-vmssfedcba   Ready    agent   13m    v1.14.1
```

Add the [official stable Helm repository][helm-stable-repo], which contains the NGINX ingress controller Helm chart.

```console
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
```

Create a Kubernetes namespace for the NGINX ingress controller and install it using `helm`.

```console
kubectl create ns nginx
helm install nginx stable/nginx-ingress --namespace nginx --version 1.27.0
```

> [!NOTE]
> The above example creates a public endpoint for your ingress controller. If you need to use a private endpoint for your ingress controller instead, add the *--set controller.service.annotations."service\\.beta\\.kubernetes\\.io/azure-load-balancer-internal"=true* parameter to the *helm install* command. For example:
> ```console
> helm install nginx stable/nginx-ingress --namespace nginx --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal"=true --version 1.27.0
> ```
> This private endpoint is exposed within the virtual network where you AKS cluster is deployed.

Get the IP address of the NGINX ingress controller service using [kubectl get][kubectl-get].

```console
kubectl get svc -n nginx --watch
```

The sample output shows the IP addresses for all the services in the *nginx* name space.

```console
NAME                                  TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
nginx-nginx-ingress-controller        LoadBalancer   10.0.19.39     <pending>        80:31314/TCP,443:30521/TCP   10s
nginx-nginx-ingress-default-backend   ClusterIP      10.0.210.231   <none>           80/TCP                       10s
...
nginx-nginx-ingress-controller        LoadBalancer   10.0.19.39     MY_EXTERNAL_IP   80:31314/TCP,443:30521/TCP   26s
```

Add an *A* record to your DNS zone with the external IP address of the NGINX service using [az network dns record-set a add-record][az-network-dns-record-set-a-add-record].

```azurecli
az network dns record-set a add-record \
    --resource-group myResourceGroup \
    --zone-name MY_CUSTOM_DOMAIN \
    --record-set-name *.nginx \
    --ipv4-address MY_EXTERNAL_IP
```

The above example adds an *A* record to the *MY_CUSTOM_DOMAIN* DNS zone.

In this article, you use the [Azure Dev Spaces Bike Sharing sample application](https://github.com/Azure/dev-spaces/tree/master/samples/BikeSharingApp) to demonstrate using Azure Dev Spaces. Clone the application from GitHub and navigate into its directory:

```cmd
git clone https://github.com/Azure/dev-spaces
cd dev-spaces/samples/BikeSharingApp/charts
```

Open [values.yaml][values-yaml] and make the following updates:
* Replace all instances of *<REPLACE_ME_WITH_HOST_SUFFIX>* with *nginx.MY_CUSTOM_DOMAIN* using your domain for *MY_CUSTOM_DOMAIN*. 
* Replace *kubernetes.io/ingress.class: traefik-azds  # Dev Spaces-specific* with *kubernetes.io/ingress.class: nginx  # Custom Ingress*. 

Below is an example of an updated `values.yaml` file:

```yaml
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

bikesharingweb:
  ingress:
    annotations:
      kubernetes.io/ingress.class: nginx  # Custom Ingress
    hosts:
      - dev.bikesharingweb.nginx.MY_CUSTOM_DOMAIN  # Assumes deployment to the 'dev' space

gateway:
  ingress:
    annotations:
      kubernetes.io/ingress.class: nginx  # Custom Ingress
    hosts:
      - dev.gateway.nginx.MY_CUSTOM_DOMAIN  # Assumes deployment to the 'dev' space
```

Save your changes and close the file.

Create the *dev* space with your sample application using `azds space select`.

```console
azds space select -n dev -y
```

Deploy the sample application using `helm install`.

```console
helm install bikesharingsampleapp . --dependency-update --namespace dev --atomic
```

The above example deploys the sample application to the *dev* namespace.

Display the URLs to access the sample application using `azds list-uris`.

```console
azds list-uris
```

The below output shows the example URLs from `azds list-uris`.

```console
Uri                                                  Status
---------------------------------------------------  ---------
http://dev.bikesharingweb.nginx.MY_CUSTOM_DOMAIN/  Available
http://dev.gateway.nginx.MY_CUSTOM_DOMAIN/         Available
```

Navigate to the *bikesharingweb* service by opening the public URL from the `azds list-uris` command. In the above example, the public URL for the *bikesharingweb* service is `http://dev.bikesharingweb.nginx.MY_CUSTOM_DOMAIN/`.

> [!NOTE]
> If you see an error page instead of the *bikesharingweb* service, verify you updated **both** the *kubernetes.io/ingress.class* annotation and the host in the *values.yaml* file.

Use the `azds space select` command to create a child space under *dev* and list the URLs to access the child dev space.

```console
azds space select -n dev/azureuser1 -y
azds list-uris
```

The below output shows the example URLs from `azds list-uris` to access the sample application in the *azureuser1* child dev space.

```console
Uri                                                  Status
---------------------------------------------------  ---------
http://azureuser1.s.dev.bikesharingweb.nginx.MY_CUSTOM_DOMAIN/  Available
http://azureuser1.s.dev.gateway.nginx.MY_CUSTOM_DOMAIN/         Available
```

Navigate to the *bikesharingweb* service in the *azureuser1* child dev space by opening the public URL from the `azds list-uris` command. In the above example, the public URL for the *bikesharingweb* service in the *azureuser1* child dev space is `http://azureuser1.s.dev.bikesharingweb.nginx.MY_CUSTOM_DOMAIN/`.

## Configure the NGINX ingress controller to use HTTPS

Use [cert-manager][cert-manager] to automate the management of the TLS certificate when configuring your NGINX ingress controller to use HTTPS. Use `helm` to install the *certmanager* chart.

```console
kubectl apply --validate=false -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.12/deploy/manifests/00-crds.yaml --namespace nginx
kubectl label namespace nginx certmanager.k8s.io/disable-validation=true
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager --namespace nginx --version v0.12.0 jetstack/cert-manager --set ingressShim.defaultIssuerName=letsencrypt --set ingressShim.defaultIssuerKind=ClusterIssuer
```

Create a `letsencrypt-clusterissuer.yaml` file and update the email field with your email address.

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: MY_EMAIL_ADDRESS
    privateKeySecretRef:
      name: letsencrypt
    solvers:
      - http01:
          ingress:
            class: nginx
```

> [!NOTE]
> For testing, there is also a [staging server][letsencrypt-staging-issuer] you can use for your *ClusterIssuer*.

Use `kubectl` to apply `letsencrypt-clusterissuer.yaml`.

```console
kubectl apply -f letsencrypt-clusterissuer.yaml --namespace nginx
```

Update [values.yaml][values-yaml] to include the details for using *cert-manager* and HTTPS. Below is an example of an updated `values.yaml` file:

```yaml
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

bikesharingweb:
  ingress:
    annotations:
      kubernetes.io/ingress.class: nginx  # Custom Ingress
      cert-manager.io/cluster-issuer: letsencrypt
    hosts:
      - dev.bikesharingweb.nginx.MY_CUSTOM_DOMAIN  # Assumes deployment to the 'dev' space
    tls:
    - hosts:
      - dev.bikesharingweb.nginx.MY_CUSTOM_DOMAIN
      secretName: dev-bikesharingweb-secret

gateway:
  ingress:
    annotations:
      kubernetes.io/ingress.class: nginx  # Custom Ingress
      cert-manager.io/cluster-issuer: letsencrypt
    hosts:
      - dev.gateway.nginx.MY_CUSTOM_DOMAIN  # Assumes deployment to the 'dev' space
    tls:
    - hosts:
      - dev.gateway.nginx.MY_CUSTOM_DOMAIN
      secretName: dev-gateway-secret
```

Upgrade the sample application using `helm`:

```console
helm upgrade bikesharingsampleapp . --namespace dev --atomic
```

Navigate to the sample application in the *dev/azureuser1* child space and notice you are redirected to use HTTPS. Also notice that the page loads, but the browser shows some errors. Opening the browser console shows the error relates to an HTTPS page trying to load HTTP resources. For example:

```console
Mixed Content: The page at 'https://azureuser1.s.dev.bikesharingweb.nginx.MY_CUSTOM_DOMAIN/devsignin' was loaded over HTTPS, but requested an insecure resource 'http://azureuser1.s.dev.gateway.nginx.MY_CUSTOM_DOMAIN/api/user/allUsers'. This request has been blocked; the content must be served over HTTPS.
```

To fix this error, update [BikeSharingWeb/azds.yaml][azds-yaml] similar to the below:

```yaml
...
    ingress:
      annotations:
        kubernetes.io/ingress.class: nginx
        cert-manager.io/cluster-issuer: letsencrypt
      hosts:
      # This expands to [space.s.][rootSpace.]bikesharingweb.<random suffix>.<region>.azds.io
      - $(spacePrefix)$(rootSpacePrefix)bikesharingweb.nginx.MY_CUSTOM_DOMAIN
      tls:
      - hosts:
        - $(spacePrefix)$(rootSpacePrefix)bikesharingweb.nginx.MY_CUSTOM_DOMAIN
        secretName: dev-bikesharingweb-secret
...
```

Update [BikeSharingWeb/package.json][package-json] with a dependency for the *url* package.

```json
{
...
    "react-responsive": "^6.0.1",
    "universal-cookie": "^3.0.7",
    "url": "0.11.0"
  },
...
```

Update the *getApiHostAsync* method in [BikeSharingWeb/lib/helpers.js][helpers-js] to use HTTPS:

```javascript
...
    getApiHostAsync: async function() {
        const apiRequest = await fetch('/api/host');
        const data = await apiRequest.json();
        
        var urlapi = require('url');
        var url = urlapi.parse(data.apiHost);

        console.log('apiHost: ' + "https://"+url.host);
        return "https://"+url.host;
    },
...
```

Navigate to the `BikeSharingWeb` directory and use `azds up` to run your updated *BikeSharingWeb* service.

```console
cd ../BikeSharingWeb/
azds up
```

Navigate to the sample application in the *dev/azureuser1* child space and notice you are redirected to use HTTPS without any errors.

## Next steps

Learn more about how Azure Dev Spaces works.

> [!div class="nextstepaction"]
> [How Azure Dev Spaces works](../how-dev-spaces-works.md)


[az-cli]: /cli/azure/install-azure-cli
[az-aks-get-credentials]: /cli/azure/aks#az-aks-get-credentials
[az-network-dns-record-set-a-add-record]: /cli/azure/network/dns/record-set/a#az-network-dns-record-set-a-add-record
[custom-domain]: ../../app-service/manage-custom-dns-buy-domain.md#buy-an-app-service-domain
[dns-zone]: ../../dns/dns-getstarted-cli.md
[azds-yaml]: https://github.com/Azure/dev-spaces/blob/master/samples/BikeSharingApp/BikeSharingWeb/azds.yaml
[azure-account-create]: https://azure.microsoft.com/free
[cert-manager]: https://cert-manager.io/
[helm-installed]: https://helm.sh/docs/intro/install/
[helm-stable-repo]: https://helm.sh/docs/intro/quickstart/#initialize-a-helm-chart-repository
[helpers-js]: https://github.com/Azure/dev-spaces/blob/master/samples/BikeSharingApp/BikeSharingWeb/lib/helpers.js#L7
[kubectl]: https://kubernetes.io/docs/user-guide/kubectl/
[kubectl-get]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#get
[letsencrypt-staging-issuer]: https://cert-manager.io/docs/configuration/acme/#creating-a-basic-acme-issuer
[package-json]: https://github.com/Azure/dev-spaces/blob/master/samples/BikeSharingApp/BikeSharingWeb/package.json
[values-yaml]: https://github.com/Azure/dev-spaces/blob/master/samples/BikeSharingApp/charts/values.yaml
