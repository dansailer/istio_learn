# Learn istio repo
Following the [Learn Istio](https://istio.io/latest/docs/examples/microservices-istio/) tutorial, gathering scripts and gists.

## Current Links (IBM Cloud)
The displayed IP and port are examples and not valid (anymore).
BookInfo: http://169.57.112.115:31213/productpage

Grafana: http://dashboard.169.57.112.115.xip.io:31213

Jaeger: http://jaeger.169.57.112.115.xip.io:31213

Prometheus: http://prometheus.169.57.112.115.xip.io:31213

Kiali: http://kiali.169.57.112.115.xip.io:31213

IBM Cloud: https://cloud.ibm.com/kubernetes/clusters


## PreReqs
You will need a running Kubernetes Cluster. Here are 3 different options. 

### Install istio-cli

```
brew install istioctl
```


### DigitalOcean

Open a new [DigitalOcean](https://cloud.digitalocean.com/kubernetes) account to have access to 100$ of promotion credit (only available in the first month) and later would cost approximately 30$/month. Create a development Kubernetes installation `k8s-digitalocean-learn-istio`

Download the kube config file from DigitalOcean and use it to connect kubectl to the new cluster as well as download the DigitalOcean cli `doctl`

```
brew install doctl
export KUBECONFIG=~/.kube/config:$(pwd)/k8s-digitalocean-learn-istio-kubeconfig.yaml
kubectl config use-context do-fra1-k8s-digitalocean-learn-istio
kubectl config current-context
```

Access the [cluster](https://cloud.digitalocean.com/kubernetes/clusters/)



### IBM Cloud free Kubernetes Cluster (30 days)
Open a [IBM Cloud](https://cloud.ibm.com/docs/containers?topic=containers-getting-started) account to have access to the free managable Kubernetes cluster and create a free 30day available Kubernetes installation `learn-istio`

```
brew install --cask ibm-cloud-cli
ibmcloud login -a cloud.ibm.com -r us-south --sso 
ibmcloud ks cluster config --cluster learn-istio
kubectl config current-context
```

Access the [cluster](https://cloud.ibm.com/kubernetes/clusters)


### Kind - local Kubernetes cluster

Before running kind, be sure to give Docker VM enough memory (~ 6 CPU and 9 GB RAM).

`Docker - Preferences - Resources`

```sh
brew install kind
kind create cluster --name learn-istio
kubectl config get-contexts
```


## Installation
Before executing these commands verify that at least one of these contexts are existing in your kube config.

```
kubectl config get-contexts
CURRENT   NAME                                   CLUSTER                                AUTHINFO                                                                             NAMESPACE
*         do-fra1-k8s-digitalocean-learn-istio   do-fra1-k8s-digitalocean-learn-istio   do-fra1-k8s-digitalocean-learn-istio-admin                                           
          kind-learn-istio                       kind-learn-istio                       kind-learn-istio                                                                     
          learn-istio/c0d7ecdd07v2gejkeca0       learn-istio/c0d7ecdd07v2gejkeca0       XXXXXXXXXXXX/iam.cloud.ibm.com-identity   default
```


Be sure to be connected to the right Kubernetes context (if you have several) for these commands to be executed at the right cluster.

```
kubectl config use-context ...
kubectl config current-context
```

### Create namespace and setup istio on namespace

```
export NAMESPACE=istio-learn-tutorial
kubectl create namespace $NAMESPACE
istioctl install -n $NAMESPACE --set profile=demo -y
kubectl label namespace $NAMESPACE istio-injection=enabled
```

### Install Demo application "BookInfo"

```
kubectl apply -n $NAMESPACE -l version!=v2,version!=v3 -f https://raw.githubusercontent.com/istio/istio/release-1.8/samples/bookinfo/platform/kube/bookinfo.yaml


kubectl get -n $NAMESPACE services
kubectl rollout -n $NAMESPACE status deployment productpage-v1
kubectl get -n $NAMESPACE pods
kubectl exec -n $NAMESPACE "$(kubectl get -n $NAMESPACE pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -s productpage:9080/productpage | grep -o "<title>.*</title>"
```

### Setup ingress gateway

```
kubectl apply -n $NAMESPACE -f https://raw.githubusercontent.com/istio/istio/release-1.8/samples/bookinfo/networking/bookinfo-gateway.yaml
istioctl analyze -n $NAMESPACE
kubectl get svc istio-ingressgateway -n istio-system
```

### Access the application via istio gateway

Depending on your Cloud Provider you will have different access to the application.

#### Digital Ocean

The Digital Ocean environment has an external load balancer that can be used for the ingress gateway.

```
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
echo "$GATEWAY_URL"
open "http://$GATEWAY_URL/productpage"
```

#### IBM Cloud

In the IBM Cloud Ingress is not supported for free clusters and so there is no external load balancer. We use Kubernetes node and node port instead to access the application.     

```
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export INGRESS_HOST=$(ibmcloud ks workers --cluster learn-istio --output json | jq -rM .[0].publicIP)
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
open "http://$GATEWAY_URL/productpage"
```

#### Kind

```
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
open "http://$GATEWAY_URL/productpage"
```


### Deploy the Kiali dashboard, along with Prometheus, Grafana, and Jaeger

Deploy the addons. Run command multiple times in case of messages such as `unable to recognize "samples/addons/kiali.yaml"`.

```
kubectl apply -f samples/addons
kubectl rollout status deployment/kiali -n istio-system
```

Access the different addons dashboards (uses port-forward internally)
```
istioctl dashboard kiali
istioctl dashboard prometheus
istioctl dashboard grafana
istioctl dashboard zipkin
istioctl dashboard jaeger
istioctl dashboard envoy
istioctl dashboard controlz
```


Continue with 


These tasks are a great place for beginners to further evaluate Istio’s features using this demo installation:
https://istio.io/latest/docs/setup/getting-started/#next-steps
https://istio.io/latest/docs/examples/microservices-istio/setup-kubernetes-cluster/

[Request routing](https://istio.io/latest/docs/tasks/traffic-management/request-routing/)
Fault injection
Traffic shifting
Querying metrics
Visualizing metrics
Accessing external services
Visualizing your mesh
Before you customize Istio for production use, see these resources:

Deployment models
Deployment best practices
Pod requirements
General installation instructions





# Scrap Ideas
### Create ingress routes for Kiali, Grafana, Tracing and Prometheus

For IBM Cloud only

```
export INGRESS_HOST=$(ibmcloud ks workers --cluster learn-istio --output json | jq -rM .[0].publicIP)

kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: istio-system
  namespace: istio-system
  annotations:
    kubernetes.io/ingress.class: istio
spec:
  rules:
  - host: dashboard.${INGRESS_HOST}.xip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          serviceName: grafana
          servicePort: 3000
  - host: jaeger.${INGRESS_HOST}.xip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          serviceName: tracing
          servicePort: 80
  - host: prometheus.${INGRESS_HOST}.xip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          serviceName: prometheus
          servicePort: 9090
  - host: kiali.${INGRESS_HOST}.xip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          serviceName: kiali
          servicePort: 20001    
EOF

kubectl get ingress istio-system -n istio-system -o jsonpath='{..host}'
```

The dasboards are now available using these DNS entries
```
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
kubectl get ingress istio-system -n istio-system -o jsonpath='{..host}' | tr ' ' '\n' | awk '{ printf "http://%s:'$INGRESS_PORT'\n", $1 }'
```
