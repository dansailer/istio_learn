# Learn istio repo
Following the [Learn Istio](https://istio.io/latest/docs/examples/microservices-istio/) tutorial, gathering scripts and gists.

## PreReqs
### Create your own Kubernetes Cluster (free) and install ibm-cloud-cli
Open a [IBM Cloud](https://cloud.ibm.com/docs/containers?topic=containers-getting-started) account to have access to the free managable Kubernetes cluster and create a free 30day available Kubernetes installation `learn-istio`


```sh
brew install --cask ibm-cloud-cli
ibmcloud login -a cloud.ibm.com -r us-south --sso 
ibmcloud ks cluster config --cluster learn-istio
kubectl config current-context
```

Access the [cluster](https://cloud.ibm.com/kubernetes/clusters)

### Install istio-cli and download samples application

```sh
brew install istioctl
curl -L -o istio-1.8.2-osx.tar.gz https://storage.googleapis.com/istio-release/releases/1.8.2/istio-1.8.2-osx.tar.gz
tar -xzvf istio-1.8.2-osx.tar.gz istio-1.8.2/samples
mv istio-1.8.2/samples .
rm -rf istio-1.8.2*
```

### Install Kind for local testing and create cluster
```sh
brew install kind
kind create cluster --name learn-istio
kubectl config get-contexts
```


## Installation
Before executing these commands be sure to be connected to the right Kubernetes context.
For IBM Cloud use

```sh
kubectl config use-context learn-istio
```

or for local Kind installation

```sh
kubectl config use-context kind-learn-istio
```

### Create namespace and setup istio on namespace

```sh
export NAMESPACE=istio-learn-tutorial
kubectl create namespace $NAMESPACE
istioctl install -n $NAMESPACE --set profile=demo -y
kubectl label namespace $NAMESPACE istio-injection=enabled
```

### Install Demo application "BookInfo"

```sh
kubectl apply -n $NAMESPACE -f samples/bookinfo/platform/kube/bookinfo.yaml

kubectl get -n $NAMESPACE services
kubectl get -n $NAMESPACE pods
kubectl exec -n $NAMESPACE "$(kubectl get -n $NAMESPACE pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -s productpage:9080/productpage | grep -o "<title>.*</title>"
```

### Setup ingress gateway

```sh
kubectl apply -n $NAMESPACE -f samples/bookinfo/networking/bookinfo-gateway.yaml
istioctl analyze -n $NAMESPACE
kubectl get svc istio-ingressgateway -n istio-system
```

### Access the application via istio gateway
The free IBM Cloud does not offer an external load balancer. We use node and node port instead to access the application.     

```sh
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export INGRESS_HOST=$(ibmcloud ks workers --cluster learn-istio-free --output json | jq -rM .[0].publicIP)
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
open "http://$GATEWAY_URL/productpage"
```