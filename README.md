# Learn istio repo
Following the [Learn Istio](https://istio.io/latest/docs/examples/microservices-istio/) tutorial, gathering scripts and gists.

## Current Links (IBM Cloud)
BookInfo: http://169.57.112.115:31213/productpage (this will change when cluster is deleted or recreated)
Grafana: http://dashboard.169.57.112.115.xip.io:31213
Jaeger: http://jaeger.169.57.112.115.xip.io:31213
Prometheus: http://prometheus.169.57.112.115.xip.io:31213
Kiali: http://kiali.169.57.112.115.xip.io:31213
IBM Cloud: https://cloud.ibm.com/kubernetes/clusters

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
Be sure to give Docker VM enough memory (~ 6GB).

`Docker - Preferences - Resources - Memory -> 6GB`

```sh
brew install kind
kind create cluster --name learn-istio
kubectl config get-contexts
```


## Installation
Before executing these commands verify that at least one of these contexts are existing in your kube config.

```
kubectl config get-contexts
CURRENT   NAME                               CLUSTER                            AUTHINFO                                                                             NAMESPACE
*         kind-learn-istio                   kind-learn-istio                   kind-learn-istio                                                                     
          learn-istio/c0d7ecdd07v2gejkeca0   learn-istio/c0d7ecdd07v2gejkeca0   XXXXXXXXXXXXX/iam.cloud.ibm.com-identity   default
```


Be sure to be connected to the right Kubernetes context for these commands to be executed at the right cluster.

For IBM Cloud use

```
ibmcloud ks cluster config  --cluster learn-istio
kubectl config current-context
```

or for local Kind installation

```
kubectl config use-context kind-learn-istio
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
kubectl apply -n $NAMESPACE -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -n $NAMESPACE -l version!=v2,version!=v3 -f https://raw.githubusercontent.com/istio/istio/release-1.8/samples/bookinfo/platform/kube/bookinfo.yaml

kubectl get -n $NAMESPACE services
kubectl rollout -n $NAMESPACE status deployment productpage-v1
kubectl get -n $NAMESPACE pods
kubectl exec -n $NAMESPACE "$(kubectl get -n $NAMESPACE pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -s productpage:9080/productpage | grep -o "<title>.*</title>"
```

### Setup ingress gateway

```
kubectl apply -n $NAMESPACE -f samples/bookinfo/networking/bookinfo-gateway.yaml
istioctl analyze -n $NAMESPACE
kubectl get svc istio-ingressgateway -n istio-system
```

### Access the application via istio gateway

Neither the free IBM Cloud (Ingress is not supported for free clusters.) nor Kind offer an external load balancer. We use Kubernetes node and node port instead to access the application.     

For IBM Cloud
```
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export INGRESS_HOST=$(ibmcloud ks workers --cluster learn-istio --output json | jq -rM .[0].publicIP)
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
open "http://$GATEWAY_URL/productpage"
```

For Kind
```
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
open "http://$GATEWAY_URL/productpage"
```


### Deploy the Kiali dashboard, along with Prometheus, Grafana, and Jaeger

Deploy the addons. Run command multiple times in case of messages such as `unable to recognize "samples/addons/kiali.yaml"`. From here on, the Kind commands are missing, as laptop is missing some power...

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






