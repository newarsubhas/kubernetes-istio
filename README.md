## 1. run this line into your machine console 

```
curl -sL https://raw.githubusercontent.com/prabhatpankaj/ubuntustarter/master/initial.sh | sh

curl -sL https://raw.githubusercontent.com/prabhatpankaj/kubernetes-onpremise-ubuntu/master/configure.sh | sh

```
## 2. Running Docker without sudo permits Running Docker with sudo all time is not a great idea. We will fix this in this step 

```
sudo usermod -aG docker ${USER}
sudo service docker restart
```
## 3. Create the cluster

At this point we create the cluster by initiating the master with kubeadm. Only do this on the master node.

```
ifconfig
```
* you should get somthing like this 

```
eth0      Link encap:Ethernet  HWaddr 02:ac:32:ae:87:20  
          inet addr:10.0.1.133  Bcast:10.0.1.255  Mask:255.255.255.0
          inet6 addr: fe80::ac:32ff:feae:8720/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:18309 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1283 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:26423737 (26.4 MB)  TX bytes:161155 (161.1 KB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:192 errors:0 dropped:0 overruns:0 frame:0
          TX packets:192 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:14456 (14.4 KB)  TX bytes:14456 (14.4 KB)
```
* sipcalc 10.0.1.133/16

```
-[ipv4 : 10.0.1.133/16] - 0

[CIDR]
Host address		- 10.0.1.133
Host address (decimal)	- 2887715554
Host address (hex)	- AC1F0AE2
Network address		- 10.1.0.0
Network mask		- 255.255.0.0
Network mask (bits)	- 16
Network mask (hex)	- FFFF0000
Broadcast address	- 10.31.255.255
Cisco wildcard		- 0.0.255.255
Addresses in network	- 65536
Network range		- 10.1.0.0 - 10.31.255.255
Usable range		- 10.1.0.1 - 10.31.255.254

-

```
* We'll now use the internal IP address to broadcast the Kubernetes API - rather than the Internet-facing address.
* You must replace --apiserver-advertise-address with the IP of your host.
```
sed -i '9s/^/Environment="KUBELET_EXTRA_ARGS=--fail-swap-on=false"\n/' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

systemctl daemon-reload

systemctl restart kubelet

kubeadm init --ignore-preflight-errors Swap --pod-network-cidr=10.0.0.0/16 --apiserver-advertise-address=10.0.1.133 --kubernetes-version v1.11.1
```

## 4. Configure an unprivileged user-account and Take a copy of the Kube config:

```
sudo useradd kubeuser -G sudo -m -s /bin/bash
sudo passwd kubeuser
sudo su kubeuser
sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
echo "export KUBECONFIG=$HOME/admin.conf" | tee -a ~/.bashrc
source ~/.bashrc

```

## 5. Make sure you note down the join token command i.e. 

```
kubeadm join 10.0.1.133:6443 --token 0daec3.ql0fin8xr87erlc2 --discovery-token-ca-cert-hash sha256:4a52b12b7953f0713c3a4f4f2084cfad9bc003da12180670a46268589eb1a9d5

```
## 6. Install networking . use Flannel or WeaveWorks
* Flannel provides a software defined network (SDN) using the Linux kernel's overlay and ipvlan modules.

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

```
* Another popular SDN offering is Weave Net by WeaveWorks.
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

```

## 7-a. Join the worker nodes to the cluster
* After finish step 6, you has been completed setup master node of your kubernetes cluster. To setup other machine to join into your cluster
* Prepare your machine as step 1
Run command kubeadm join with params is the secret key of your kubernetes cluser and your master node ip as STEP 4


## 7-b. Allow a single-host cluster
* Kubernetes is about multi-host clustering - so by default containers cannot run on master nodes in the cluster. Since we only have one node - we'll taint it so that it can run containers for us.
 ```
kubectl taint nodes --all node-role.kubernetes.io/master-
 ```

## 8. get cluster

```
kubectl get all --namespace=kube-system
```
## 9. install helm
```
curl -Lo /tmp/helm-linux-amd64.tar.gz https://kubernetes-helm.storage.googleapis.com/helm-v2.9.0-linux-amd64.tar.gz
tar -xvf /tmp/helm-linux-amd64.tar.gz -C /tmp/
chmod +x  /tmp/linux-amd64/helm && sudo mv /tmp/linux-amd64/helm /usr/local/bin/

```
## 10. install istio 

```
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.0.0 sh 

cd istio-1.0.0

echo "export PATH="$PATH:$PWD/bin"" | tee -a ~/.bashrc

source ~/.bashrc

```
### 10. Configure Istio CRD
* Istio has extended Kubernetes via Custom Resource Definitions (CRD). Deploy the extensions by applying crds.yaml.

```

kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml -n istio-system

```
## 11. Install Istio with default mutual TLS authentication
* To Install Istio and enforce mutual TLS authentication by default, use the yaml istio-demo-auth.yaml

```
kubectl apply -f install/kubernetes/istio-demo-auth.yaml

```
## 12. Verify istio (wait untill STATUS become Running/Completed )
```
kubectl get pods -n istio-system
kubectl get svc -n istio-system

```
* Istio Architecture
The previous step deployed the Istio Pilot, Mixer, Ingress-Controller, and Egress-Controller, and the Istio CA (Certificate Authority).

* Pilot - 
Responsible for configuring the Envoy and Mixer at runtime.

* Envoy - 
Sidecar proxies per microservice to handle ingress/egress traffic between services in the cluster and from a service to external services. The proxies form a secure microservice mesh providing a rich set of functions like discovery, rich layer-7 routing, circuit breakers, policy enforcement and telemetry recording/reporting functions.

* Mixer - 
Create a portability layer on top of infrastructure backends. Enforce policies such as ACLs, rate limits, quotas, authentication, request tracing and telemetry collection at an infrastructure level.

* Ingress/Egress - 
Configure path based routing.

* Istio CA - 
Secures service to service communication over TLS. Providing a key management system to automate key and certificate generation, distribution, rotation, and revocation

* The overall architecture is shown below.

![](/images/istio-arch.png)

## 13. install sample project (https://github.com/istio/istio/tree/master/samples/bookinfo)

* When deploying an application that will be extended via Istio, the Kubernetes YAML definitions are extended via kube-inject. This will configure the services proxy sidecar (Envoy), Mixers, Certificates and Init Containers.

* create custom resource services (Tiller, servicegraph, grafana, jaeger, prometheus)
* edit externalIPs

```
cd kubernetes-istio

kubectl apply -f customresourceservices.yaml

```

```

cd istio-1.0.0

kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)

kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

kubectl get pods

```
## 14. Confirm the gateway has been created:
```
kubectl get gateway
```
## 15. determine INGRESS_HOST ,INGRESS_PORT and SECURE_INGRESS_PORT
```
export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o 'jsonpath={.items[0].status.hostIP}')

export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')

```
## 16. Set GATEWAY_URL:
```
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```
## 18. Apply default destination rules

```
kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml

```
## User Based Testing / Request Routing
* The example below will send all traffic for the user "jason" to the reviews:v2, meaning they'll only see the black stars.

```
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml

```
## Traffic Shaping for Canary Releases
* The rule below ensures that 50% of the traffic goes to reviews:v1 (no stars), or reviews:v3 (red stars).
```
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml

```

## New Releases
* Given the above approach, if the canary release were successful then we'd want to move 100% of the traffic to reviews:v3.
```
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml

```

## List All Routes
* It's possible to get a list of all the rules applied using 

```
istioctl get virtualservices
istioctl get virtualservices -o yaml
```
## Generate Load
* Make view the graphs, there first needs to be some traffic. Execute the command below to send requests to the application.

```
while true; do
  curl -s http://${GATEWAY_URL}/productpage > /dev/null
  echo -n .;
  sleep 0.2
done
```
## Access Dashboards
* Grafana

```
http://YourPublicIP:3000/dashboard/db/istio-mesh-dashboard
```
