# k8s-team-ckad-training
k8s-team-ckad-training



# Requirements

## Docker 
[https://docs.docker.com/install/#supported-platforms](https://docs.docker.com/install/#supported-platforms)

## Kubectl 

[https://kubernetes.io/docs/tasks/tools/install-kubectl/](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## kind (Kubernetes in docker)

On Mac via Homebrew:

```console
brew install kind
```

On Windows:

```powershell
curl.exe -Lo kind-windows-amd64.exe https://github.com/kubernetes-sigs/kind/releases/download/v0.7.0/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe c:\some-dir-in-your-PATH\kind.exe

OR via [Chocolatey](https://chocolatey.org/packages/kind)
choco install kind
```

# Install kind cluster 

On Mac via Homebrew:
```sh
brew install kind
```

On Windows:
```
curl.exe -Lo kind-windows-amd64.exe https://github.com/kubernetes-sigs/kind/releases/download/v0.7.0/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe c:\some-dir-in-your-PATH\kind.exe

# OR via Chocolatey (https://chocolatey.org/packages/kind)
choco install kind
```


# Create a kind cluster

```sh
cat <<EOF > /tmp/cluster-config.txt 
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true 
  podSubnet: 192.168.0.0/16 
EOF

kind create cluster --config /tmp/cluster-config.txt 
```

Deploy calico overlay network (required for the network policy)

```
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
kubectl -n kube-system set env daemonset/calico-node FELIX_IGNORELOOSERPF=true
```

Make sure all pods are up and Running before you follow the next steps.


```sh
$kubectl get pods --all-namespaces -o wide

NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE     IP             NODE                 NOMINATED NODE   READINESS GATES
kube-system          calico-kube-controllers-5c45f5bd9f-dtx4s     1/1     Running   0          2m31s   192.168.82.1   kind-control-plane   <none>           <none>
kube-system          calico-node-b79hl                            1/1     Running   0          2m30s   172.17.0.2     kind-control-plane   <none>           <none>
kube-system          coredns-6955765f44-68xf6                     1/1     Running   0          2m31s   192.168.82.2   kind-control-plane   <none>           <none>
kube-system          coredns-6955765f44-dt8sx                     1/1     Running   0          2m30s   192.168.82.4   kind-control-plane   <none>           <none>
kube-system          etcd-kind-control-plane                      1/1     Running   0          2m43s   172.17.0.2     kind-control-plane   <none>           <none>
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          2m43s   172.17.0.2     kind-control-plane   <none>           <none>
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          2m43s   172.17.0.2     kind-control-plane   <none>           <none>
kube-system          kube-proxy-bknls                             1/1     Running   0          2m30s   172.17.0.2     kind-control-plane   <none>           <none>
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          2m43s   172.17.0.2     kind-control-plane   <none>           <none>
```

# Services and Networking (13%)

https://kubernetes.io/docs/concepts/services-networking/service/

https://kubernetes.io/docs/concepts/services-networking/network-policies/

Create namespace nginx and set is as the current namespace

```sh
kubectl create ns nginx
kubectl config set-context --current --namespace=nginx 
```

# Deploy a nginx pod
```sh
cat <<EOF | kubectl create -f - 
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: my-nginx
  name: nginx
  namespace: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
EOF
````

List all pods with label `app=my-nginx`
```
kubectl get pods -l app=my-nginx
```

create service with expose command

```
kubectl expose po nginx --port=80 --type=NodePort
kubectl get svc -o wide
```

export cluster ip to a variable
```
CLUSTER_IP=`kubectl get svc  -o jsonpath='{.items[0].spec.clusterIP}'`
echo $CLUSTER_IP
```

# Network policies 

create a temporary pod to connect to the nginx nodeport (leave this command running in a new console tab).

```
kubectl run curl --image=curlimages/curl --restart=Never -it --rm -- sh -c "while true; do curl --connect-timeout 3 -I $CLUSTER_IP:80 && sleep 1 ; done"
```

Wait until the previous command returns http status 200 (OK). 

Create a network policy to deny all ingress and egress traffic in the current namespace. 
```
cat <<EOF | kubectl apply -f - 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: nginx
spec:
  podSelector: {}   
  policyTypes:
  - Ingress
  - Egress
EOF
```

Open the previous tab, where the curl pod command is running, you will probably see a curl error like `curl: (28) Connection timed out after 3001 milliseconds`. This means the network policy is in place and all the inbound/outbound traffic in the namespace is denied. 

Create a new network policy to allow egress traffic to port 80 and 443.

```
cat <<EOF | kubectl apply -f - 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-http
  namespace: nginx
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
    - protocol: UDP
      port: 53
EOF
```

Create another network policy to allow ingress traffic from pod with label `run=curl` to port 80 and 443.

```
cat <<EOF | kubectl apply -f - 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-http-from-curlpod
  namespace: nginx
spec:
  policyTypes:
  - Ingress
  podSelector: {}
  ingress:
  - from:    
    - podSelector:
        matchLabels:
          run: curl
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443        
EOF
```

# Dns

https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/

Create a nslookup pod 
```
kubectl run nslookup --image=curlimages/curl --restart=Never -it --rm sh
```

As you can see, k8s will preprend all dns queries with `nginx.svc.cluster.local svc.cluster.local cluster.local`
```
cat /etc/resolv.conf
search nginx.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

Resolve nginx svc 

```
nslookup nginx.nginx.svc.cluster.local
Server:		10.96.0.10
Address:	10.96.0.10:53

Name:	nginx.nginx.svc.cluster.local
Address: 10.96.204.245```


```(⎈ |kind-kind:nginx)➜  ~ k get svc -n nginx
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.96.204.245   <none>        80:30679/TCP   55m
(⎈ |kind-kind:nginx)➜  ~
````
