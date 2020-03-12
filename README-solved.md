# k8s-team-ckad-training

## State Persistence

https://kubernetes.io/docs/concepts/storage/volumes/#emptydir​

https://kubernetes.io/docs/concepts/storage/persistent-volumes/​

https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/


### EmptyDir
- Scratch space​
- Checkpoint a long computation for recovery​
- Log processing​
- Fetch files from content manager for web container

<details><summary>show</summary>

```yaml

apiVersion: v1​
kind: Pod​
metadata:​
  name: test-pd​
spec:​
  containers:​
  - image: k8s.gcr.io/test-webserver​
    name: test-container​
    volumeMounts:​
    - mountPath: /cache​
      name: cache-volume​
  volumes:​
  - name: cache-volume​
    emptyDir: {}
```

</details>


### Persistent Volumes
- Piece of storage in the cluster​
- Provisioned with Storage classes​
- Lifecycle independent of pods​

<details><summary>show</summary>

```yaml
kind: PersistentVolume​
apiVersion: v1​
metadata:​
  name: my-pv​
spec:​
  storageClassName: local-storage​
  capacity:​
    storage: 1Gi​
  accessModes:​
    - ReadWriteOnce​
  hostPath:​
    path: "/mnt/data"
```
</details>



### Persistent Volumes Claims
- Request for storage by a user​
- Pods comsume node resources, PVC consume PV resources​
- Claims can request specific size, acess mode and storage class


<details><summary>show</summary>

```yaml
apiVersion: v1​
kind: PersistentVolumeClaim​
metadata:​
  name: my-pvc​
spec:​
  storageClassName: local-storage​
  accessModes:​
    - ReadWriteOnce​
  resources:​
    requests:​
      storage: 512Mi
```

```yaml
kind: Pod​
apiVersion: v1​
metadata:​
  name: my-pvc-pod​
spec:​
  containers:​
  - name: busybox​
    image: busybox​
    command: ["/bin/sh", "-c", "while true; do sleep 3600; done"]​
    volumeMounts:​
    - mountPath: "/mnt/storage"​
      name: my-storage​
  volumes:​
  - name: my-storage​
    persistentVolumeClaim:​
      claimName: my-pvc
```

</details>



### Exercises

### Lab-01

- Create a "busybox" pod with a simple emptyDir volume, named "my-volume", to the container at the path "/tmp/storage"​


<details><summary>show</summary>

```yaml
apiVersion: v1​
kind: Pod​
metadata:​
  name: volume-pod​
spec:​
  containers:​
  - image: busybox​
    name: busybox​
    command: ["/bin/sh", "-c", "while true; do sleep 3600; done"]​
    volumeMounts:​
    - mountPath: /tmp/storage​
      name: my-volume​
  volumes:​
  - name: my-volume​
    emptyDir: {}​
```
</details>



### Lab-02

- Create a "hostPath" type persistent volume named "mysql-pv", of size "1Gi" and access mode "ReadWriteOnce"​

<details><summary>show</summary>

```yaml
apiVersion: v1​
kind: PersistentVolume​
metadata:​
  name: mysql-pv​
spec:​
  storageClassName: localdisk​
  capacity:​
    storage: 1Gi​
    accessModes:​
      - ReadWriteOnce ​
  hostPath: ​
    path: "/mnt/data"​
```
​</details>


- Create a persistent volume claim named "mysql-pv-claim", of size "500Mi", access mode "ReadWriteOnce" and storage class name "localdisk"

<details><summary>show</summary>

```yaml
apiVersion: v1​
kind: PersistentVolumeClaim​
metadata:​
  name: mysql-pv-claim​
spec:​
  storageClassName: localdisk​
  accessModes:​
    - ReadWriteOnce​
  resources:​
    requests:​
      storage: 500Mi
```

</details>


## Configurations

- Understand ConfigMaps​
https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/​

- Understand SecurityContexts​
https://kubernetes.io/docs/tasks/configure-pod-container/security-context/​

- Define an application's resource requirements​
https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/​

- Create and consume secrets​
https://kubernetes.io/docs/concepts/configuration/secret/​

- Understand service accounts​
https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/


### Exercises

### Lab-01
- Create a configmap in the namespace sbp called sbp-config with literal value appname=philis​
- Verify that the configmap we just created is correct.​
- Delete the configmap we just created

<details><summary>show</summary>


```sh
# kubectl -n sbp create cm sbp-config --from-literal=appname=philis​
# kubectl -n sbp get cm sbp-config -o yaml;      ​
# kubectl -n sbp describe cm sbp-config​
# kubectl -n sbp delete cm sbp-config​
```
</details>


### Lab-02
- Create a configmap with two values key1=schuberg and key2=philis in the namespace sbp named sbp-config and verify that configmap is created correctly​
- Create an nginx pod and load environment values from the above configmap sbp-config and exec into the pod and verify the environment variables and delete the pod

<details><summary>show</summary>

```sh
# cat >> config.txt << EOF​
key1=schuberg​
key2=philis​
EOF​

# kubectl -n sbp create configmap sbp-config --from-file=config.txt​
# kubectl -n sbp create configmap sbp-config --from-literal=key1=value1 --from-literal=key2=value2​
# kubectl -n sbp get cm sbp-config -o yaml​
# kubectl run nginx --image=nginx --restart=Never --dry-run -o yaml > nginx-pod.yml​
```

- Edit the yml to below file and create​

```yaml
apiVersion: v1​
kind: Pod​
metadata:​
  creationTimestamp: null​
  labels:​
    run: nginx​
  name: nginx​
spec:​
  containers:​
  - image: nginx​
    name: nginx​
    resources: {}​
    envFrom:​
    - configMapRef:​
        name: sbp-config​
  dnsPolicy: ClusterFirst​
  restartPolicy: Never​
status: {}​
```

```sh
# kubectl -n sbp create -f nginx-pod.yml​
# kubectl -n sbp exec -it nginx -- env​
#kubectl -n sbp delete po nginx​
```
</details>


### Lab-03
- Create an env file sbp.env with var1=val1 and create a configmap sbp-config from this env file and verify the configmap​
- Create an nginx pod and load environment values from the above configmap envcfgmap and exec into the pod and verify the environment variables and delete the pod

<details><summary>show</summary>

```sh
# echo var1=val1 > sbp.env​
# cat sbp.env​
# kubectl -n sbp create configmap sbp-config --from-env-file=sbp.env​
# kubectl -n sbp get configmap sbp-config -o yaml --export​
# kubectl run nginx --image=nginx --restart=Never --dry-run -o yaml > nginx-pod.yml​
```

- Edit the yml to below file and create​

```yaml
apiVersion: v1​
kind: Pod​
metadata:​
  creationTimestamp: null​
  labels:​
    run: nginx​
  name: nginx​
spec:​
  containers:​
  - image: nginx​
    name: nginx​
    resources: {}​
    env:​
    - name: ENVIRONMENT​
      valueFrom:​
        configMapKeyRef:​
          name: sbp-config​
          key: var1​
  dnsPolicy: ClusterFirst​
  restartPolicy: Never​
status: {}​
```

```sh
# kubectl create -f nginx-pod.yml​
# kubectl exec -it nginx -- env​
# kubectl delete po nginx​
```

</details>



### Lab-04
- Create a configmap called sbp-config in the namespace sbp with values var1=schuberg, var2=philis and create an nginx pod with volume nginx-volume which reads data from this configmap sbp-config and put it on the path /etc/cfg​
- Create a pod called sbp-sleep with the image busybox which executes command sleep 3600 and makes sure any Containers in the Pod, all processes run with user ID 1000 and with group id 2000 and verify.​
- Create the same pod as above this time set the securityContext for the container as well and verify that the securityContext of container overrides the Pod level securityContext.


<details><summary>show</summary>

```sh
# kubectl config set-context --current --namespace=sbp​
# kubectl create cm sbp-config --from-literal=var1=schuberg --from-literal=var2=philis​
# kubectl describe cm sbp-config​
```

- Create the nginx pod with volume​

```yaml
apiVersion: v1​
kind: Pod​
metadata:​
  creationTimestamp: null​
  labels:​
    run: nginx​
  name: nginx​
spec:​
  volumes:​
  - name: nginx-volume​
    configMap:​
      name: sbp-config​
  containers:​
  - image: nginx​
    name: nginx​
    resources: {}​
    volumeMounts:​
    - name: nginx-volume​
      mountPath: /etc/cfg​
  dnsPolicy: ClusterFirst​
  restartPolicy: Never​
status: {}​
```

```sh
# kubectl create -f nginx-volume.yml​
# kubectl exec -it nginx -- /bin/sh​
# cd /etc/cfg​
# ls​
# kubectl run sbp-sleep --image=busybox --restart=Never --dry-run -o yaml -- /bin/sh -c "sleep 3600;" > sbp-sleep.yml​
```

- Edit the pod like below and create​

 ​
```yaml
apiVersion: v1​
kind: Pod​
metadata:​
  creationTimestamp: null​
  labels:​
    run: sbp-sleep​
  name: sbp-sleep​
spec:​
  securityContext: # add security context​
    runAsUser: 1000​
    runAsGroup: 2000​
  containers:​
  - args:​
    - /bin/sh​
    - -c​
    - sleep 3600;​
    image: busybox​
    name: sbp-sleep​
    resources: {}​
  dnsPolicy: ClusterFirst​
  restartPolicy: Never​
status: {}​
```
 
```sh 
# kubectl create -f sbp-sleep.yml​
# kubectl exec -it sbp-sleep – sh id ​
```

- Create the same pod as above this time set the securityContext for the container as well and verify that the securityContext of container overrides the Pod level securityContext.​

 ```yaml
apiVersion: v1​
kind: Pod​
metadata:​
  creationTimestamp: null​
  labels:​
    run: sbp-sleep​
  name: sbp-sleep​
spec:​
  securityContext:​
    runAsUser: 1000​
  containers:​
  - args:​
    - /bin/sh​
    - -c
    - sleep 3600;​
    image: busybox​
    securityContext:​
      runAsUser: 2000​
    name: sbp-sleep​
    resources: {}​
  dnsPolicy: ClusterFirst​
  restartPolicy: Never​
status: {}​
```

```sh
# kubectl create -f sbp-sleep.yml​
# kubectl exec -it sbp-sleep -- sh id ​
```
</details>



### Lab-05
- Create a Pod nginx and specify a memory request and a memory limit of 100Gi and 200Gi respectively which is too big for the nodes and verify pod fails to start because of insufficient memory​
- Create a secret mysecret with values user=myuser and password=mypassword​
- Create an nginx pod which reads username as the environment variable​
- Create a service account called admin
- Create a busybox pod which executes this command sleep 3600 with the service account admin and verify

<details><summary>show</summary>

```sh
# kubectl run nginx --image=nginx --restart=Never --dry-run -o yaml > nginx.yml​
```

- Add the resources section and create​

```yaml
apiVersion: v1​
kind: Pod​
metadata:​
  creationTimestamp: null​
  labels:​
    run: nginx​
  name: nginx​
spec:​
  containers:​
  - image: nginx​
    name: nginx​
    resources:​
      requests:​
        memory: "100Gi"​
        cpu: "0.5"​
      limits:​
        memory: "200Gi"​
        cpu: "1"​
  dnsPolicy: ClusterFirst​
  restartPolicy: Never​
status: {}​
```

```sh
# kubectl create -f nginx.yml​
# kubectl describe po nginx ​
```
​
- Create a secret mysecret with values user=myuser and password=mypassword​

```sh
#kubectl create secret generic my-secret --from-literal=username=user --from-literal=password=mypassword​
# kubectl run nginx --image=nginx --restart=Never --dry-run -o yaml > nginx.yml​
```

- Add env section below and create​

```yaml
apiVersion: v1​
kind: Pod​
metadata:​
  creationTimestamp: null​
  labels:​
    run: nginx​
  name: nginx​
spec:​
  containers:​
  - image: nginx​
    name: nginx​
    env:​
    - name: USER_NAME​
      valueFrom:​
        secretKeyRef:​
          name: my-secret​
          key: username​
    resources: {}​
  dnsPolicy: ClusterFirst​
  restartPolicy: Never​
status: {}​
```​

```sh
# kubectl create -f nginx.yml​
# kubectl exec -it nginx -- env​
# kubectl create sa admin​
# kubectl run busybox --image=busybox --restart=Never --dry-run -o yaml -- /bin/sh -c "sleep 3600" > busybox.yml​
# kubectl create -f busybox.yml​
# kubectl describe po busybox​
```

```yaml
apiVersion: v1​
kind: Pod​
metadata:​
  creationTimestamp: null​
  labels:​
    run: busybox​
  name: busybox​
spec:​
  serviceAccountName: admin​
  containers:​
  - args:​
    - /bin/sh​
    - -c​
    - sleep 3600​
    image: busybox​
    name: busybox​
    resources: {}​
  dnsPolicy: ClusterFirst​
  restartPolicy: Never​
status: {}​
```
</details>




## Observability 

- Understand LivenessProbes and ReadinessProbes​
https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes​

- Understand Container Logging​
https://kubernetes.io/docs/concepts/cluster-administration/logging/​

- Understand how to monitor applications in kubernetes​
https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application/​

- Understand Debugging in Kubernetes​
https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application/

### Exercises

### Lab-01
- Create an nginx pod with a liveness probe that just runs the command 'ls'. ​
- Save its YAML in pod.yaml. ​
- Run it, check its probe status, delete it.

<details><summary>show</summary>

```sh
# kubectl run nginx --image=nginx --restart=Never --dry-run -o yaml > pod.yaml​
# kubectl create -f pod.yaml​
# kubectl describe pod nginx | grep -i liveness
# kubectl delete -f pod.yaml​
```
</details>

### Lab-02
- Modify the pod.yaml file so that liveness probe starts kicking in after 5 seconds whereas the interval between probes would be 5 seconds. ​
- Run it, check the probe, delete it.

<details><summary>show</summary>
```sh
# kubectl explain pod.spec.containers.livenessProbe
# kubectl create -f pod.yaml​
# kubectl describe po nginx | grep -i liveness​
# kubectl delete -f pod.yaml​
```
</details>



### Lab-03
- Create an nginx pod (that includes port 80) with an HTTP readinessProbe on path '/' on port 80. ​
- Again, run it, check the readinessProbe, delete it.

<details><summary>show</summary>
```sh
# kubectl run nginx --image=nginx --dry-run -o yaml --restart=Never --port=80 > pod.yaml​
# vim pod.yaml​
# kubectl create -f pod.yaml​
# kubectl describe pod nginx | grep -i readiness
# kubectl delete -f pod.yaml​
```
</details>



### Lab-04
- Create a busybox pod that runs:​
 ```sh
 'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done' ​
 ```
- Check its logs


<details><summary>show</summary>
```sh
# kubectl run busybox --image=busybox --restart=Never -- /bin/sh -c 'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done'​
​# kubectl logs busybox -f
```
</details>



### Lab-05
- Create a busybox pod with this command “echo I am from busybox pod; sleep 3600;”​
- Verify the logs.

<details><summary>show</summary>
```sh
# kubectl run busybox --image=busybox --restart=Never -- /bin/sh -c "echo I am from busybox pod; sleep 3600;"​
# kubectl logs busybox
```
</details>


### Lab-06
- Create a busybox pod that runs:​

```sh
'ls /notexist'​
```
- Determine if there's an error, see it. ​
- In the end, delete the pod

<details><summary>show</summary>
```sh
# kubectl run busybox --restart=Never --image=busybox -- /bin/sh -c 'ls /notexist'
# kubectl logs busybox​
# kubectl describe po busybox​
# kubectl delete po busybox
```
</details>


### Lab-07
- Create the pod with this kubectl create -f  not-running.yml. ​
- The pod is not in the running state. ​
- Debug it.

<details><summary>show</summary>
```sh
# kubectl create -f not-running.yml ​
# kubectl get pod not-running​
# kubectl describe po not-running
# kubectl edit pod not-running
### OR
# kubectl set image pod/not-running not-running=nginx​
```
</details>


### Lab-08
- Create a busybox pod that runs 'notexist'. ​
- Determine if there's an error, see it.​
- In the end, delete the pod forcefully with a 0 grace period

<details><summary>show</summary>
```sh
# kubectl run busybox --restart=Never --image=busybox -- notexist​
# kubectl logs busybox # will bring nothing! container never started​
# kubectl describe po busybox # in the events section, you'll see the error​
# kubectl get events | grep -i error # you'll see the error here as well​
# kubectl delete po busybox --force --grace-period=0​
```
</details>

### Lab-09
- Get CPU/memory utilization for nodes​

<details><summary>show</summary>
```sh
# kubectl top nodes​
```
</details>

### Lab-10
- Get the memory and CPU usage of all the pods and find out top 3 pods which have the highest usage​
- Put them into the cpu-usage.txt file

<details><summary>show</summary>
```sh
# kubectl top pod --all-namespaces | sort --reverse --key 3 --numeric | head -3​
# kubectl top pod --all-namespaces | sort --reverse --key 3 --numeric | head -3 > cpu-usage.txt​
# cat cpu-usage.txt
```
</details>



## Services & Networking

https://kubernetes.io/docs/concepts/services-networking/service/​

https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/​

https://kubernetes.io/docs/concepts/services-networking/network-policies/

### Exercises

### Lab-01

- Create an abstraction  layer infront of the pods​
- Use labels to select which pods to send traffic to​


<details><summary>show</summary>

```yaml
apiVersion: v1​
kind: Service​
metadata:​
  name: my-service​
spec:​
  type: ClusterIP​
  selector:​
    app: nginx​
  ports:​
  - protocol: TCP​
    port: 8080​
    targetPort: 80​
```
</details>



### Lab-02

- Network Policies:​
    - How a group of pods can communication with:​
        - Another group of pods​
        - Other network endpoints​

- Supports configuring:​
    - Ingress/Egress rules based on:​
        - CIDR blocks, exceptions​
        - Namespace and pod selector based on labels​
        - Ports/protocols

<details><summary>show</summary>

```yaml
apiVersion: networking.k8s.io/v1​
kind: NetworkPolicy​
metadata:​
  name: test-network-policy​
  namespace: default​
spec:​
  podSelector:​
    matchLabels:​
      role: db​
  policyTypes:​
  - Ingress​
  - Egress​
  spec:​
  ingress:​
  - from:​
    - ipBlock:​
        cidr: 172.17.0.0/16​
        except:​
        - 172.17.1.0/24​
    - namespaceSelector:​
        matchLabels:​
          project: myproject​
    - podSelector:​
        matchLabels:​
          role: frontend​
    ports:​
    - protocol: TCP​
      port: 6379​
      spec:​
        egress:​
        - to:​
          - ipBlock:​
            cidr: 10.0.0.0/24​
          ports:​
          - protocol: TCP​
          port: 5978
```

</details>
