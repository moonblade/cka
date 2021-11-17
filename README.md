

## Todo

- go thorugh kubernetes.io docs for better higher level overview understanding
- ~~lxc containers~~, mac only has lxc client
- [x] k8s the hard way
- [x] RBAC
- Go through all concepts in k8s.io

### Curriculum


##### Storage - 10%
- [x] Understand storage classes, persistent volumes
- [x] Understand volume mode, access modes and reclaim policies for volumes
- [x] Understand persistent volume claims primitive
- [x] Know how to configure applications with persistent storage

##### Troubleshooting - 30%
- [x] Evaluate cluster and node logging
- [x] Understand how to monitor applications
- [x] Manage container stdout & stderr logs
- [x] Troubleshoot application failure
- [x] Troubleshoot cluster component failure
- [x] Troubleshoot networking

##### Workloads & Scheduling - 15%
- [x] Understand deployments and how to perform rolling update and rollbacks
- [x] Use ConfigMaps and Secrets to configure applications
- [x] Know how to scale applications
- [x] Understand the primitives used to create robust, self-healing, application deployments
- [x] Understand how resource limits can affect Pod scheduling
- [x] Awareness of manifest management and common templating tools

##### Cluster Architecture, Installation & Configuration - 25%
- [x] Manage role based access control (RBAC)
- [x] Use Kubeadm to install a basic cluster
- [x] Manage a highly-available Kubernetes cluster
- [x] Provision underlying infrastructure to deploy a Kubernetes cluster
- [x] Perform a version upgrade on a Kubernetes cluster using Kubeadm
- [x] Implement etcd backup and restore

##### Services & Networking - 20%
- [x] Understand host networking configuration on the cluster nodes
- [x] Understand connectivity between Pods
- [x] Understand ClusterIP, NodePort, LoadBalancer service types and endpoints
- [x] Know how to use Ingress controllers and Ingress resources
- [x] Know how to configure and use CoreDNS
- [x] Choose an appropriate container network interface plugin

### Notes

##### Cluster Architecture, Installation & Configuration

<details>
<summary>
RBAC
</summary>

- kube-apiserver -authorization=stuff,RBAC --otherstuff
- apiVersion: rbac.authorization.k8s.io/v1
- bindings are immutable, to update, need to destroy and recreate, so use `kubectl auth reconcile -f my-rbac-rules.yaml --dry-run=client`. 
Note: might be that its just role ref that is immutable and subjects can be changed
- create role binding manually `kubectl create rolebinding ...` 
- clusteroles can be aggregated, with aggregationRule 
- `k api-resources -o wide` to find the resource names and associated verbs for roles
- service accounts need name, can set it to be in namespace as well
- to use a service account, do `k get secrets <service account secret name> -o json | jq .data.token | cut -f2 -d\" | base64 -d`, then create a user and context in kubeconfig 
with 

```
- context: mind
   cluster: existing-one
   user: newsvcone
  name: itsname
... users section
- name: newsvcone
  user:
    token: copied token
```

---
Create a new user

```
openssl genrsa -out mb.key 2048
openssl req -key mb.key -out mb.csr -new
```

pass through k8s csr 

```
cat > csr.yaml <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: usersign
spec:
  usages:
    - client auth
  signerName: kubernetes.io/kube-apiserver-client
  request: $(cat mb.csr | base64)
  username: test2
EOF
k apply -f csr.yaml
```

approve it
```
k get csr
k certificate approve csr usersign
```

get it out
```
k get csr usersign -o json | jq .status.certificate | tr -d \" | base64 -d  > mb.crt
```

make a config out of it
```
k config view --flatten --minify > kconfig
k config set-credentials test2 --client-certificate=mb.crt --client-key=mb.key --username test2 --embed-certs --kubeconfig=kconfig 
k config set-context test2 --cluster=kind-kind --user=test2 --kubeconfig=kconfig
```

test it
```
k get po --kubeconfig=kconfig
```

test it again after rolebinding
```
k create role test2r --resource=pods --verb="*"
k create rolebinding test2rb --user=test2 --role=test2r
k get po --kubeconfig=kconfig
```
</details>

<details><summary> kubeadm </summary>

setup
```
kubeadm init --apiserver-advertise-address=$(hostname -i) --kubenetes-version <version>
kubeadm token list

kubeadm join ip --token <token>
```

Haha, don't make me laugh, if it were that simple everyone would be doing it.
first you gotta install dockerd
```
apt install docker.io
```
but dockerd, being a son of a bitch, says, nope, I dont want no truck with systemd, I'll just be here using cgroupfs as by cgroup driver. That can't go on, set it right
Fucking issue is that this thing seems to be available nowhere apart from stackoverflow, how the heck are you supposed to do this in a test environment.
```
cat > /etc/docker/daemon.json << EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
sudo systemctl restart docker
```

now that thats done can safely install kubelet and kubeadm and stuff, just make sure that swap is off
```
swapoff -a
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Then fucking finally you can do the first commands to set it up

OR, get this, throw docker out of the window, like the expired piece of hot garbage it is and make your bed with containerd, and best of all, it installs with apt
```
apt install containerd
```
no muss, no fuss. You might have to do a `modprobe br_netfilter`, but thats easy, since its there in the k8s.io install kubeadm section
From there its the same schenanigans of installing and setting up the control plane

### Multi master setup

Created digital ocean droplet for controller-1, 2 and node-1

ran
```
modprobe br_netfilter
apt install -y containerd
swapoff -a
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

echo 1 > /proc/sys/net/ipv4/ip_forward

kubeadm init --control-plane-endpoint $loadbalancerip:443 --apiserver-advertise-address $selfip --pod-network-cidr=10.200.0.0/16 --ignore-preflight-errors=NumCPU,Mem

kubeadm init phase upload-certs --upload-certs

kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

Opened scheduler conf, and removed port=0 line
```
vim /etc/kubernetes/manifests/kube-scheduler.yaml
systemctl restart kubelet
```

Add cni
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

to join a node, ran the command from above, and used that
```
kubeadm token create --print-join-command
```

Get into a catch22 situation that you can't do upload-certs without load balancer actively working, and load balancer wont work till the cluster is up.
So end up giving up on it, and instead try the init phase upload-certs later, but it does'nt work either. So finally just give up and move the certs by hand.

</details>

<details>
<summary>etcd</summary>

install etcd client
```
apt install etcd-client
```

create a cm to test
```
kubectl create cm  test --from-literal=key1=config1 --from-literal=key2=config2
```

take backup
```
ETCDCTL_API=3 etcdctl --endpoints localhost:2379 \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  snapshot save backup
```

delete cm 
```
kubectl delete cm test
kubectl get cm
```

to restore, need to kill the pod, then delete the etcd drive, then run restore
```
kubectl delete po etcd-kind-control-plane -n kube-system
rm -rf /var/lib/etcd
ETCDCTL_API=3 etcdctl --endpoints localhost:2379 \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --data-dir=/var/lib/etcd \
  snapshot restore backup
kubectl get cm
```


</details>

##### Workloads & Scheduling

<details>
<summary>Deployment</summary>

Deployment has been made super simple from its predecessors by kubernetes. Used to be that you needed to use replicasets or replicatsetcontroller to do an update. 
Now all that is done natively with deployment objects, just update the image version and tada it just gets updated as rolling update, not even having any downtime. Baby proof, Like fuck, how cool is that.

to undo a rollout, just go through the rollout history and pick the one you need.

```
k create deploy web --image=nginx:1.14.1 --replicas=10 --dry-run=client -o=yaml > web.yaml
k apply -f web.yaml
k set image deploy/web --record nginx=nginx:1.16.1 --record
k rollout history deploy/web
k rollout undo deploy/web 
# k rollout undo deploy/web --to-revision 3
# k rollout pause/resume/restart deploy/[name]
```

Similary scaling is baby proofed as well, for manual scaling, add the --scale param or `.spec.replicas`, change it to scale to more pods.
```
kubectl scale deployment.v1.apps/nginx-deployment --replicas=10
```

For auto scaling, run the autoscale command with min, max and cpu-percent
Needs metrics server though, its install is also present in deployment page. Takes a while to get started, enough time to make you questions if its working or not, and try installing other shit on it.
```
k autoscale deploy/web --min=1 --max=10 --cpu-percent=80
```

</details>

<details><summary>Manifests</summary>
Manifests are the yaml files that will be used to create the k8s objects, and this will be pushed to version control.
But when you whave multiple clusters, say dev and prod, and you want different configurations for each, it becomes a hassle to either maintain two versions or change each time.

So templating engines do the hard work.

- kustomize.
Its inbuilt into kubectl at this point, with -k in kubectl apply. create a kuztumization yaml in how you want it changed and then run k apply.
It will also hash the changes and change the name of the resource so that the pod will get restarted and wont be using the old stale one.

- helm
more of a package manager than templater, but has template engine uses with external values file. 

- yq
similar to jq, commandline modifications to values just before k8s apply is done. 

- Other code based approach
Create objects from some other code to create the yaml then apply those.

</details>

<details> <summary> Configmaps and secrets </summary>

```
k create cm -h

k create cm literalcm --from-literal foo=bar

k create cm envcm --from-env-file .env --dry-run=client -o yaml > envcm.yaml

k create cm propcm --from-file a.properties --dry-run=client -o yaml  > propcm.yaml
```

The configmap page in k8s.io has a good sample with almost everything (except configmap everything as .env) (can get it from `k explain --recursive cm | more > podspc.yaml`)

env valueFrom to take each values from .env

envFrom configmapRef to take everything from .env

volumemount to use as full files instead, its always readOnly, removing items will mount everything.

```
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # Define the environment variable
        - name: PLAYER_INITIAL_LIVES # Notice that the case is different here
                                     # from the key name in the ConfigMap.
          valueFrom:
            configMapKeyRef:
              name: game-demo           # The ConfigMap this value comes from.
              key: player_initial_lives # The key to fetch.
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      envFrom:
        - configMapRef:
            name: test-cm
        - secretRef:
            name: firstsecret
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    # You set volumes at the Pod level, then mount them into containers inside that Pod
    - name: config
      configMap:
        # Provide the name of the ConfigMap you want to mount.
        name: game-demo
        # An array of keys from the ConfigMap to create as files
        items:
        - key: "game.properties"
          path: "game.properties"
        - key: "user-interface.properties"
          path: "user-interface.properties"
# to test it
k exec -it po/configmap-demo-pod -- sh
```

##### Secrets

Pretty much the same as configmaps, just encrypted. if using data field, its base64 encoded, if its stringData, its normal

Normal secrets are of the type Opaque (generic), but specialized once like tls for certs can be used as well.


```
k create secret generic firstsecret --from-literal=foo=bar
k create secret generic secondsecret --from-file secretFile
k create secret generic thirdsecret --from-env-file secretEnv
```

```
apiVersion: v1
kind: Secret
metadata:
  name: secret-basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: t0p-Secret
```

```
apiVersion: v1
kind: Secret
metadata:
  name: secret-dockercfg
type: kubernetes.io/dockercfg
data:
  extra: YmFyCg==
  .dockercfg: |
        "<base64 encoded ~/.dockercfg file>"
```

yaml with usage of it
```
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # Define the environment variable
        - name: PLAYER_INITIAL_LIVES # Notice that the case is different here
                                     # from the key name in the ConfigMap.
          valueFrom:
            configMapKeyRef:
              name: envcm           # The ConfigMap this value comes from.
              key: testkey # The key to fetch.
      envFrom:
        - configMapRef:
            name: secondenvcm
        - secretRef:
            name: firstsecret
        - secretRef:
            name: thirdsecret
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
      - name: secret
        mountPath: "/secret"
        readOnly: true
  volumes:
    # You set volumes at the Pod level, then mount them into containers inside that Pod
    - name: config
      configMap:
        # Provide the name of the ConfigMap you want to mount.
        name: propcm
    - name: secret
      secret:
        secretName: secondsecret
```

its mounted in `tmpfs` in the node and mounted to pod, so wont be in disk.

</details>

<details><summary> Robustness </summary>

- Deploy has replicasets which maintain the number of replicas. If a pod is killed another takes its place, 
if a node dies, after five minutes of buffer time, all pods in that node are recreated. If it comes back, the pods in the node are removed. Meanwhile they goto unknown state. 

- pdb, pod disruption budget, to setup max unavailable or min available number of pods, selects with label selectors 

`k create pdb test-pdb --selector run=nginx --min-available=50%`

- [k8s.io - readiness, liveness, startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
  - liveness probe, know if the pod is active and traffic can be routed there its container property, and available in `k explain deploy --recursive > deployspec`
  - liveness - pod is alive and doesn't need restart
  - readiness - pod can receive traffic
  - startup - same as liveness, but runs on start of pod since first liveness might take a long time 

</details>

<details><summary> Resource limits and sheduling </summary>

###### Limits and requests

- limits can be setup with limits - memory and cpu, if it goes above the limits the pod will be evicted  
- Request is the amount necessary to allocate a pod, wont allocate a pod with lower amount of resources left

```
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
 name: redis
 labels:
   name: redis-deployment
   app: example-voting-app
spec:
 replicas: 1
 selector:
   matchLabels:
    name: redis
    role: redisdb
    app: example-voting-app
 template:
   spec:
     containers:
       - name: redis
         image: redis:5.0.3-alpine
         resources:
           limits:
             memory: 600Mi
             cpu: 1
           requests:
             memory: 300Mi
             cpu: 500m
       - name: busybox
         image: busybox:1.28
         resources:
           limits:
             memory: 200Mi
             cpu: 300m
           requests:
             memory: 100Mi
             cpu: 100m
```

###### Resource quotas

Set at namespace level by admin to limit available resources to a namespace.
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-example
spec:
  hard:
    requests.cpu: 2
    requests.memory: 2Gi
    limits.cpu: 3
    limits.memory: 4Gi
```

```
k create quota -h
```

###### Limit ranges
To set default quotas and limits for pods

```
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:
      cpu: 1
    defaultRequest:
      cpu: 0.5
    type: Container
```

</details>

##### Services and networking

<details> <summary> host networking and pod connectivity </summary>

###### host networking

- ip per pod,
- pods can find each other without nat, since each have its own (mostly cidr) ip
- node agents (kubelet, kube proxy) can talk to all pods in the node
- service creates virtual ip for a set of pods

###### pod to pod

![pod to pod](https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/pod-to-pod-same-node.gif)


- each pod has a veth pair, for same node pod to pod, it goes through a vitual bridge. 
For different node, bridge doesn't find address, so goes to host, then finds the pod to node entry (cidr iptable) and gets to correct node and then to correct pod.

![pod to pod different node](https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/pod-to-pod-different-nodes.gif)

- [understanding kubernetes networking model](https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/)
- IPVS (ip virtual server), built on top of netfilter, doesn't need to setup iptables for everything, uses hashmaps so has much better scalability. 
(existing way is to add each pod in service to iptable and load balancing is done by host)

- cni is used to automate the networking process, on how to connect network namespace and network plugin
  - CNI - [weave net](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)

</details>

<details>
<summary> Service types </summary>

- ClusterIP

  - Gives an internal ip to the service available within the cluster, can forward based on ip or service name, and gets redirected to one of the pods. 
  - Ensure that selector is correct or it will not route it correctly.
  - port - svc port, targetPort - pod port

- NodePort
  - Binds to port on ALL the nodes, good for host requirements, or even outside network through firewall config. 

- loadbalancer
  - Creates load balancer on the cloud provider and gives ip to outside the cluster

</details>

<details> <summary> CoreDNS </summary>
 - setup by default from k8s 1.11, 

get config map that controls it
 ```
 k get cm -n kube-system coredns  
 ```

```
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

- change cluster.local to change zone,
- https://coredns.io/2017/05/08/custom-dns-entries-for-kubernetes/
- to rewrite, can use rewrite plugin `rewrite name foo.example.com foo.default.svc.cluster.local` for `foo` svc in `default` namespace
- for pod it would be ip.namespace

</details>

##### Storage

<details> <summary> storage </summary>

- https://kubernetes.io/docs/concepts/storage/persistent-volumes/#reserving-a-persistentvolume
- Storage classes: Storage classes are used by pvc to interact with the underlying storage, so each cloud provider would have their own storage classes 
- Persistent volume: Virtual volume plugins that are provisioned independant of pod lifecycles and be plugged in as a volume and be used as storage. Created using some underlying storage class.
- Persistent volume claim: Request by user for a specific pv. (pods use node resources, pvc's use pv resources)

- VolumeMode 
  - FileSystem - default - mounted into pods as a directory, if backing device is empty, creates file system on it before mounting
  - Block - raw block device - pod needs to know how to handle it itself.

- PV - access modes - https://stackoverflow.com/questions/37649541/kubernetes-persistent-volume-accessmode
  - ReadWriteOnce - read write mount by single node
  - ReadWriteMany - read write by multiple nodes
  - ReadOnlyMany - read write by many nodes

- Reclaim policy
  - Retain - manually reclaim
  - Recycle - does scrub (rm -rf volume/\*)
  - Delete - associated volume (by the clouod provider) is deleted


###### Creating pvc
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolumeclaim
- Remove storageClassName for default

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

</details>

##### Troubleshooting

<details> <summary> Logging </summary>

- https://kubernetes.io/docs/concepts/cluster-administration/logging/
- pod level
  - `k logs podName` to get logs for pod. 

- node level
  - A container engine handles and redirects any output generated to a containerized application's stdout and stderr streams. For example, the Docker container engine redirects those two streams to a logging driver, which is configured in Kubernetes to write to a file in JSON format.
  - Log rotation is not handled by k8s, deployment tool should do that
  - When using cri, kubelet does the rotation
  - `k logs` gets data from this log file
  - `k logs` only gets latest file, so if recently rotated, it will be empty
  - system stuff (kubelet, cri) will log to journald, if not present, will write to `/var/log`

- cluster level
  - no native solutions provided, options are
  - Use a node-level logging agent that runs on every node.
    - a daemonset on each node that takes the node logs from log folder and puts it to cloud 
  - Include a dedicated sidecar container for logging in an application pod.
    - sidecar either streams to stdout (if pods dont log to stdout) - this will make use of logging agent already in the cluster
    - or sidecar runs logging agent to pick from pod and put in cloud 
  - write logs directly to cloud from inside pod

- cluster level - centralized
  - log aggregation given by third party
  - fluentd - daemon set on each pod - log forwarder/aggregator 
  - elk - elastic, logstash, kibana  
    - kibana for ui
    - logstash to aggregate
    - elastic for store

- Audit logs
  - Everything done on a cluster is logged as audit trails
  - https://kubernetes.io/docs/tasks/debug-application-cluster/audit/
  

##### Monitoring

- `k top` to get top resource utilizing nodes or pods
- monitoring is basically making sure that you have enough resources avaialble and if a node/pod goes down there is enough space to put the remainder (and that another node can be upscaled) 
- prometheus can natively monitor k8s and nodes 
- [k8s dashbaord](https://github.com/kubernetes/dashboard) also provides monitoring data
- `kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml`, run `k proxy` to get access to dashboard on locahost:8081

- metrics server is needed for `k top`. `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml` (this is needed for pod autoscaler as well)

##### container stdout/err
- container stdout is auto piped to log files by docker to `/var/log/containers` and symlinked in `/var/log/pods`
- can get logs from pod level with `k logs`

</details>

<details>
<summary> Troubleshooting apps </summary>

### Debugging pods

describe will have list of events and state of pod
```
k describe pod podname
k get events
```

- stuck in pending state 
  - Not enough resources in node
  - hostPort, can only have as many hostports as nodes, most times service is all you need
- Stuck in waiting
  - could be failed to pull images
- pod crashes
  - get logs to see application logs to see if its issue with app itself
- pod doing something else
  - delete and create with --validate, will find unkonwn keys
  - `k get pod podname > test.yaml` and compare against the yaml you created with

```
k run 
k exec
```
these can be used to run a container in parallel or exec into it to debug stuff directly

```
k debug node/nodename -it --name=ubuntu
```
will get you acess to basically the entire node

### Debugging replication controllers 
```
k describe rc rcname
```

### Services
```
k get endpoints svcname
```
check that endpoints are made for the service

- no endpoints
  - probably selector is wonky, `k get pods --selector=foo=bar,test2=val`
- check that service exists
  - `k get svc`
  - inside a pod try pinging it `wget -O- svcname`, or `nslookup svcname`, `cat /etc/resolve.conf`
- check that service is connected to correct port
- is kueb proxy running on node, in node `ps auxw | grep kube-proxy`


</details>

<details>
<summary> Troubleshooting clusters </summary>

- check if nodes are all correct `k get no`
- get overall health of cluster, deatiled `kubectl cluster-info dump`
- check logs of services, for systemd, use `journalctl -xefu servicename` `u` is needed for service, `x` for helpful messages, `f` for follow and `e` to scroll to end
- for non systemd, check log files in `/var/log/kube*.log`

</details>

<details>
<summary> Troubleshootnetworking </summary>

- ip forwarding is off 
  - symtpom: connections time out on trying to connect to different service
  - fix: `sysctl net.ipv4.ip_forward` should be 1, if not `sysctl -w net.ipv4.ip_forward=1`

- bridge netfilter is disabled
  - check: `sysctl net.bridge.bridge-nf-call-iptables`  should be 1, if not `modprobe br_netfilter; sysctl -w net.bridge.bridge-nf-call-iptables=1`

- `iptables-save | grep serviceName` to find firewall rules for the service on node


</details>

### Logs

> Day 23 - 17 Nov, Wednesday
- Pay for exam, not scheduled yet. Realize that you overpaid but try not to get you down.
- Go through the rest of installation and configuration and test etcd restore as well.

> Day 22 - 16 Nov, Tuesday
- Start recapping stuff, do rbac code examples and add bookmarks to chrome. 
- Do kubeadm init for single cluster with katakoda

> Day 21 - 15 Nov, Monday
- Start on networking troubleshooting

> Break for reasons

> Day 20 - 13 Oct, Wednesday
- Take it slow so as to not have more burnout, and read a bit on troubleshooting clusters in the evening

> Day 19 - 12 Oct, Tuesday
- Learn about debugging

> 7 - 11 Oct, off sick

> Day 18 - 6 Oct, Wednesday
- Monitoring mostly was about metrics, so gave a cursory glance and moved on.
- Container and stdout and err, didn't find many resources on it, but generally got the idea that it was done by docker. So left it at that. 

> Day 17 - 5 Oct, Tuesday
- Take a crack at monitoring, even though its an easy thing, spent most of the time distracted and didn't get shit moving.

> Day 16 - 4 Oct, Monday
- Start on logging, need to give more attention here as its worth a ton. So should take some time on it. 

> Day 15 - 3 Oct, Sunday
- Try to do storage object tasks, realize its not worth the effort, and just read on storage classes, and just decide to create pv and pvc to use on a pod.
- Go through the rest of them like butter since its really only one thing and not multiple, pvc and its associated stuff. So just finish the whole module.

> Day 14 - 2 Oct, Saturday
- Go through coreDNS, realize everythings a fractal and you need to cut off somewhere and stop at configuring basics.
- Go through CNI, and just learn install of cni, which amounts to appying a yaml file, and call the module done

> Day 13 - 1 Oct, Friday
- Read up on ingress, assume its easy, realize that ingress controllers are way different from what you thought they were and more similar to cni (third party stuff) and that ingress resources are also different.
Will come back to this when I'm done with storages so I can try a storagebucket resource ingress.

> Day 12 - 30 Sep, Thursday
- Read up on host and pod networking, not from implementation point, but mostly as to know how its organized
- Learn about the service types. Go through iterations of issues of not copying ip correctly, and not passing correct selector finally setting up services correctly.

> Day 11 - 29 Sep, Wednesday
- Knock another easy topic out of the way, resource limitting on pods.

> Day 10 - 28 Sep, Tuesday
- Pay the price for binging series till 2 and not wake up for learning shit in the morning, have a terrible day, and try to complete understanding primitives for robustness by the end of the day. 

> Day 9 - 27 Sep, Monday
- Start on configmaps and secrets, and create and test configmaps and secrets, find it relatively easy and realize that it was just tiredness keeping me from doing it yesterday.

> Day 8 - 26 Sep, Sunday
- Get frustrated with other hardware projects because hardware is an iffy bitch and figure might as well do some k8s learning.
- Create a k8s cluster on digital ocean for rollout/deployment test.
- Realize that deployments are baby proof and updating it involves literally changing one line, rollout history and undo is similarly easy as well. 
Which makes sense, Its solving the problem it aims to solve.
- Feel like wasting the day, and study an easy portion of manifests instead to feel like something was done.

> Day 7 - 25 Sep, Saturday
- Realize that its probably stupid to pay and provision systems in all the web based locations, so instead just read up on it, its mostly the same but each would need its own cli app, which is pointless to try to learn. 
- Try to upgrade kubernets version on a cluster, but realize that you're getting stuck with the install portion of it. Finally try it enough times that it works. Then do the upgrade part way too easily. 

> Day 6 - 24 Sep
- Do etcd backup and restore again, wtih delete /var/lib/etcd and restore with --data-dir and restoring somewhere else and editing /etc/kubernetes/manifests/etcd.yaml mount dir
- Realize that you've already forgotten how to create user, so go back on your own notes and find the k8s.io link where its given and learn that instead. hopefully.
- Take the rest of the day off realizing you're in a haze already.

> Day 5 - 23 Sep
- Think you're going to take a break and focus on other projects and not burn up, realize that other project has bugs and go right back in
- Try to setup multi master cluster with --upload-certs, try your damndest to make it work, finally give up on it, and move certs on your own to create multi master cluster.
- Think you're going to sleep at normal times like 10, and then stay up till 1 am, figuring out random bugs in etcd restore

> Day 4 - 22 Sep
- Check sample question for rbac, realize its easy to the point that you're overpreparing, and move onto kubeadm
- Created a new cluster with kubeadm, breezed through it, tried to create a multi master cluster, didn't have much luck due to not having a load balancer and couldn't figure out how to create one myself
So going to try multi cluster one with digital ocean
- Get ass kicked by the humble kubeadm, wrangling an eel might have been easier. Anyway, docker is setup with cgroupfs instead of systemd, and kubelet doesn't start, whether or not its systemd and kubeadmin doesn't notice if it starts anyway
- Say fuck it, do it again, and emerge with flying colors, realize how good and how bad life can get with a few simple tweaks to a few config files. 
Spout some inane bullshit like "its the mistakes the guide you towards learning" and then be on your way to the actual goal of multi master setup

> Day 3 - 21 Sep
- Reading about users, setup some rbac basic roles and bindings,
- tried setting up service account and adding role bindings to it, need to try it again and log it. 
- create new users, sign them with cluster admin cert and then use that to create a new kubeconfig for a user. 
Didn't particularly like the process, feels like if there is no external AD, k8s should get into user management as well, even if its a niche thing 

> Day 2 - 20 sep

- Completed k8s the hard way, had issue with setting up firewall, didn't know about cidr, so when it gave issue for the internal new ip spaces for pdos, removed that part from config and used vpc internal ips. 
That came to bite me in the ass later, so removed all the resources and started again. with the correct firewall config. Once completed, ran the smoke tests and brought down the stack.
- Read [how kubernetes certificates work](https://jvns.ca/blog/2017/08/05/how-kubernetes-certificates-work/)

> Day 1 - 19 sep

- Read "the kubernetes book" by nigel poulton
- Setup 3 nodes, and added a deployment for nginx, its service with a nodeport, and accessed it with curl
- Realize that lxc doesn't support mac, and find alternatives for k8s hard mode.
- Started 4 digital ocean instances, one as host and the others as cluster nodes with one master
- Got stuck on the k8s the hard way repo since I got confused about setting up loadbalancers. Thought whether or not those were required for the controller only or for everyone. Will get it up tomorrow.


