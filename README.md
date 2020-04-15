# ISTIO Virtualservice routing rules
We are going to:
 * setup a *minikube* cluster
 * install *istio* 
 * install an application 

## PURPOSE 
The purpose of the test is to demonstrate the flexibility of ISTIO routing rules.  
We want to:
 * reach *app v2* if we point to  xp.example.com/version 
 * reach *app v1* if we point to www.example.com/version
 * reach *app v3* if we point to www.example.com/ or xp.example.com/


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
  * custom python flask API application (3 versions: backend-v1 , backend-v2, backend-v3)  
    This application read the env variable *VERSION* and return it when called.
  * service (ClusterIP) to expose the application (label: app: backend)
* istio-gw.yaml
  * define an istio gateway named *demo-gateway* that route traffic from the external world to the inside of k8s
* istio-backend.yaml
  * defines an istio VirtualService that route traffic to the instances of the python app
  * defines an istio DestinationRule to distinguish between the 2 versions of the python app  
    (labels: version: v1 , v2 v3)

## Checking out features 
Get and set env variables
```bash
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
export INGRESS_HOST=$(minikube -p istio-mk ip)
```

Check if the app respond on base url (should see app v3):
```bash
while true; do curl http://$INGRESS_HOST:$INGRESS_PORT -H "Host: www.example.com"; sleep .2;done
# AND 
while true; do curl http://$INGRESS_HOST:$INGRESS_PORT -H "Host: xp.example.com"; sleep .2;done
```

Check if routing for /version work as expected:
 * xp.example.com/version  ---> app v2
 * www.example.com/version ---> app v1
```bash
while true; do curl http://$INGRESS_HOST:$INGRESS_PORT/version -H "Host: www.example.com"; sleep .2;done
# OR
while true; do curl http://$INGRESS_HOST:$INGRESS_PORT/version -H "Host: xp.example.com"; sleep .2;done
```

## Clean up
```bash
minikube -p istio-mk stop
minikube -p istio-mk delete
```

