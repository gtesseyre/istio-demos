This is a set of instructions intended to walk someone through a demo of Istio components and features. It is largely inspired from the tasks on istio.io and some Envoy material. 
This is a constant work in progress and as new versions get released this demo might change or get embellished/enhanced with time :)
It is targeted to the presenter who will talk over it which explain the raw format with minimal information.

I am assuming some familiarities with Google Cloud as I am using Google Kubernetes Engine (GKE) to run Kubernetes and Istio in this example. 

Some of the variables in this example include <GCP_ZONE>, <PROJECT_ID>, <POD_NAME>, <INGRESS_IP>, <ISTIO_MIXER_POD_NAME>, those would depend on your own environment and you would have to replace these values with the actual ones in the following instructions

### 0 - Initialize gcloud environment
In order to select the account and project in which we will run the demo
```
gcloud init
```
## 1 - Install Kubernetes and Istio
### 1.a - Create GKE cluster
Create the Kubernetes Cluster (I am using a standard default configuration here)
```
gcloud container clusters create istio-demo \
    --cluster-version=1.9.4-gke.1 \
    --zone <GCP_ZONE> \
    --project <PROJECT_ID>
```
Get your credentials (it adds the info for this newly created cluster to ~/.kube/config), to be able to interact with this cluster from your laptop through kubectl
```
gcloud container clusters get-credentials istio-demo \
    --zone <GCP_ZONE> \
    --project <PROJECT_ID>
```
Grant cluster admin permissions to the current user for RBAC access later on during Istio install
```
kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=$(gcloud config get-value core/account)
```

### 1.b - Install Istio base 
Get the latest version
```
curl -L https://git.io/getLatestIstio | sh -
```
```
cd istio-0.7.1
```
Add this into your path to use istioctl
```
export PATH=$PWD/bin:$PATH
```
Install Istio (with mTLS enabled in that case, use istio.yaml instead if you don't want mTLS)
```
kubectl apply -f install/kubernetes/istio-auth.yaml
```

### 1.c - Verify Istio base 
Check the services and pods in the istio-system namespace
```
kubectl get svc -n istio-system
```
```
kubectl get pods -n istio-system
```
### 1.d - Install the sidecar injector
Create certificate/key pair
```
./install/kubernetes/webhook-create-signed-cert.sh \
    --service istio-sidecar-injector \
    --namespace istio-system \
    --secret sidecar-injector-certs
```
Install the sidecar injector Configmap
```
kubectl apply -f install/kubernetes/istio-sidecar-injector-configmap-release.yaml
```
Set the caBundle in the webhook install YAML
```
cat install/kubernetes/istio-sidecar-injector.yaml | \
     ./install/kubernetes/webhook-patch-ca-bundle.sh > \
     install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml
```
Deploy the sidecar injector
```
kubectl apply -f install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml
```
Label the default namespace (that we will use to deploy our app later) to enable automatic sidecar injection
```
kubectl label namespace default istio-injection=enabled
```
### 1.e - Verify the sidecar injector 
```
kubectl -n istio-system get deployment -listio=sidecar-injector
```
```
kubectl get namespace -L istio-injection
```

## 2 - Deploy an application
We will use the BookInfo application in this example
```
kubectl apply -f samples/bookinfo/kube/bookinfo.yaml
```
Check the pod/svc deployment. Check the number of containers to verify the sidecar injection
```
kubectl get services
kubectl get pods
```
Get the Ingress Public IP and save it in an ENV variable for later use
```
kubectl get ingress -o wide
export GATEWAY_URL=<INGRESS_IP>:80
```
Verify connectivity
```
curl -o /dev/null -s -w "%{http_code}\n" http://${GATEWAY_URL}/productpage
```
Verify that no Istio route rules are applied currently
```
istioctl get routerules
```
## 3 - Install monitoring/visualization/tracing tools 
We will deploy Prometheus (monitoring), Grafana (dashboard), Zipkin (tracing) and Service Graph and configure port-forwarding in order to access them localy from our laptop and show these different components

Install Prometheus and configure port-forwarding
```
kubectl apply -f install/kubernetes/addons/prometheus.yaml
```
```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090 &
```
Access Prometheus locally
```
http://localhost:9090/graph
```
Install Grafana and configure port-forwarding
```
kubectl apply -f install/kubernetes/addons/grafana.yaml
```
```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
```
Access Grafana Dashboards locally
```
http://localhost:3000/dashboard/db/istio-dashboard
```
At that point it is possible to walk through the different dashboards configured out of the box (Istio / Mixer / Pilot)

Install Zipkin and configure port-forwarding
```
kubectl apply -f install/kubernetes/addons/zipkin.yaml
```
```
kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=zipkin -o jsonpath='{.items[0].metadata.name}') 9411:9411 &
```
Access Zipkin locally
```
http://localhost:9411
```
Install Service Graph and configure port-forwarding
```
kubectl apply -f install/kubernetes/addons/servicegraph.yaml
```
```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=servicegraph -o jsonpath='{.items[0].metadata.name}') 8088:8088 &   
```
Access Service Graph locally
```
http://localhost:8088/force/forcegraph.html 
```
## 4 - Generate continuous traffic 
In order to have constant usage of the app, I just create a continuous loop to query the productpage of the BookInfo application. That will come handy for the visualization of the different metrics, dashboards, traces, and traffic management rules enforcement. 
```
while true; do curl -o /dev/null -s -w "%{http_code}\n" http://${GATEWAY_URL}/productpage; done
```

## 5 - Create some routing rules 
The idea here is to create some traffic splitting, traffic shifting and delay introduction rules and notice the impact in the different tools providing the observability (Dashboards, Zipkin, Service Graph) and using the BookInfo UI

All traffic to reviews V1 
```
istioctl create -f samples/bookinfo/kube/route-rule-all-v1.yaml
istioctl get routerules -o yaml
```
All traffic for the user Jason to reviews V2 
```
istioctl create -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml
istioctl get routerule reviews-test-v2 -o yaml
```
Introducing delay for user Jason when calling the rating service
```
istioctl create -f samples/bookinfo/kube/route-rule-ratings-test-delay.yaml
istioctl get routerule ratings-test-delay -o yaml
```
Deleting the rule for user Jason and reviews V2
```
istioctl delete routerule reviews-test-v2
```
Updating the routing rules with a 50/50 split between V1 and V3 for the reviews service
```
istioctl replace -f samples/bookinfo/kube/route-rule-reviews-50-v3.yaml
```
```
istioctl get routerule reviews-default -o yaml
```
Updating the routing rules with a 100% of the traffic going to V3 of the reviews service

```
istioctl replace -f samples/bookinfo/kube/route-rule-reviews-v3.yaml
```
```
istioctl get routerule reviews-default -o yaml
```
## 6 - Taking a closer look at Envoy
Get the pod name for the productpage service
```
kubectl get pods -l app=productpage
```
Check the description of the pod and highlight the certicate distribution through the use of K8S secrets
```
kubectl describe pods <POD_NAME>
```
Execute a bash inside the Envoy container, running inside the productpage pod identified previously
```
kubectl exec -it <POD_NAME> -c istio-proxy /bin/bash
```
Check the processes running inside that container (pilot agent & envoy)
```
ps -ef
```
Check the config file received by the Envoy proxy from Pilot
```
more /etc/istio/proxy/envoy-rev0.json
```
Check the automatic distribution of the certificate/key pair 
```
ls /etc/certs
```
Exit the container
```
exit 
```
Redirect Envoy's API locally
```
kubectl port-forward <POD NAME> 15000:15000 &
```
Check the stats through the Envoy API (Using a browser is also a good visual way to show this)
```
curl -s http://localhost:15000/stats | more 
```
Check the number of retrys in the stats exposed through the API (Using a browser is also a good visual way to show this)
```
curl -s http://localhost:15000/stats | grep retry
```
Check the routing information through the Envoy API (Using a browser is also a good visual way to show this)
```
curl -s http://localhost:15000/routes | more 
```

## 7 - mTLS (Optional)
Show the configuration of mTLS and the presence of the CA
```
kubectl get deploy -l istio=istio-ca -n istio-system
kubectl get configmap istio -o yaml -n istio-system | grep authPolicy | head -1
```
For the next steps you need the debug version of the Envoy proxy in order to use curl to simulate calls between services. In my case, manually updating the proxy did not work, probably because of the automatic sidecar injection precedence, so I basically had to remove the application and the sidecar injection in order to manually inject the debug version of the proxy on a newly deployed version of the app. I went through the following steps and also deleted the remaining of the previous routing rules
```
istioctl delete -f samples/bookinfo/kube/route-rule-all-v1.yaml
istioctl delete -f samples/bookinfo/kube/route-rule-ratings-test-delay.yaml
kubectl delete -f samples/bookinfo/kube/bookinfo.yaml
kubectl delete -f install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml
```
Re-deployed the BookInfo app and manually injected the debug version of the proxy
```
kubectl apply -f samples/bookinfo/kube/bookinfo.yaml
kubectl apply -f <(istioctl kube-inject --debug -f samples/bookinfo/kube/bookinfo.yaml)
```
Get the pod name for the productpage service
```
kubectl get pods -l app=productpage
```
Execute a bash inside the Envoy container, running inside the productpage pod identified previously
```
kubectl exec -it <POD_NAME> -c istio-proxy /bin/bash
```
Check the automatic distribution of the certificate/key pair 
```
ls /etc/certs
```
Try to call the details service (this should throw an SSL error)
```
curl https://details:9080/details/0
```
Now, try the same call, using the certificate/key pair provisioned by Istio as the Envoy proxy would do it 
```
curl https://details:9080/details/0 -v --key /etc/certs/key.pem --cert /etc/certs/cert-chain.pem --cacert /etc/certs/root-cert.pem -k
```
## 8 - Miscellanous - Various commands/cheatsheet
When poking at Envoy you can also display the IP tables rules that allow the interception/redirection of the traffic, you can take a look at the processes listening in the proxy, or if you're running the debug version (with curl) you can query the API locally
```
netstat -nlp
netstat -anlp
sudo iptables -t nat -v -L
curl localhost:15000/stats | grep retry
curl localhost:15000/routes 
```
A couple interesting commands for Mixer logging and redirection
```
kubectl  port-forward -n istio-system <ISTIO_MIXER_POD_NAME> 9093:9093
curl localhost:9093/metrics
kubectl logs -n istio-system <ISTIO_MIXER_POD_NAME> -c mixer
```
## 9 - Cleanup
Generic cleanup commands based on the instructions we just went through
```
istioctl delete -f samples/bookinfo/kube/route-rule-all-v1.yaml
istioctl delete -f samples/bookinfo/kube/route-rule-ratings-test-delay.yaml

kubectl delete -f samples/bookinfo/kube/bookinfo.yaml

kubectl delete -f install/kubernetes/addons/prometheus.yaml
kubectl delete -f install/kubernetes/addons/grafana.yaml
kubectl delete -f install/kubernetes/addons/zipkin.yaml
kubectl delete -f install/kubernetes/addons/servicegraph.yaml

killall kubectl 

gcloud container clusters delete istio-demo \
    --zone <GCP_ZONE> \
    --project <PROJECT_ID>
```
