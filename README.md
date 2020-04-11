# ISTIO Virtualservice rules
We are going to setup a *minikube* cluster, install *istio* and install an application to test Virtualservices capabilities.


## Environment setup
Setup minikube with the appropriate resources:
```bash
minikube start -p istio-mk --memory=8192 --cpus=3 \
  --kubernetes-version=v1.17.0 \
  --vm-driver=virtualbox --disk-size=20g
```

## ISTIO setup
```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.*
export PATH=$PWD/bin:$PATH
cd ..

## The demo configuration profile is not suitable for performance evaluation. 
## It is designed to showcase Istio functionality with high levels of tracing and access logging
## For a production setup consider: $ istioctl manifest apply

istioctl manifest apply --set profile=demo
```

## Configure sidecar injector 
Setup the automatic sidecar injection feature of istio for the *default* namespace:  
```bash
kubectl label namespace default istio-injection=enabled
```
Check
```bash
kubectl describe ns default
```

## Deploy example app & istio gw, vs and dr
```bash
kubectl apply -f deployment/backend.yaml
kubectl apply -f deployment/istio-backend.yaml
kubectl apply -f deployment/istio-gw.yaml
```
Quick explanation:  
* backend.yaml
  * custom python flask API application (2 versions: backend-v1 & backend-v2)  
    This application read the env variable *VERSION* and return it when called.
  * service (ClusterIP) to expose the application (label: app: backend)
* istio-gw.yaml
  * define an istio gateway named *demo-gateway* that route traffic from the external world to the inside of k8s
* istio-backend.yaml
  * defines an istio VirtualService that route traffic to the instances of the python app
  * defines an istio DestinationRule to distinguish between the 2 versions of the python app  
    (labels: version: v1 & v2)

## Checking out features (NOT WORKING AS EXPECTED)
Get and set env variables
```bash
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
export INGRESS_HOST=$(minikube -p istio-mk ip)
```

Check if the app respond on base url (should only go to if Host is "www.example.com" v1 and to v2 host is "xp.example.com"):
```bash
while true; do curl http://$INGRESS_HOST:$INGRESS_PORT -H "Host: www.example.com"; sleep .2;done
# OR
while true; do curl http://$INGRESS_HOST:$INGRESS_PORT -H "Host: xp.example.com"; sleep .2;done
```

## Clean up
```bash
minikube -p istio-mk stop
minikube -p istio-mk delete
```

