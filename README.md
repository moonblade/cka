

## Todo

- go thorugh kubernetes.io docs for better higher level overview understanding
- ~~lxc containers~~, mac only has lxc client
- [x] k8s the hard way
- [ ] RBAC

### Curriculum


##### Storage - 10%
- Understand storage classes, persistent volumes
- Understand volume mode, access modes and reclaim policies for volumes
- Understand persistent volume claims primitive
- Know how to configure applications with persistent storage

##### Troubleshooting - 30%
- Evaluate cluster and node logging
- Understand how to monitor applications
- Manage container stdout & stderr logs
- Troubleshoot application failure
- Troubleshoot cluster component failure
- Troubleshoot networking

##### Workloads & Scheduling - 15%
- [x] Understand deployments and how to perform rolling update and rollbacks
- Use ConfigMaps and Secrets to configure applications
- [x] Know how to scale applications
- Understand the primitives used to create robust, self-healing, application deployments
- Understand how resource limits can affect Pod scheduling
- [x] Awareness of manifest management and common templating tools

##### Cluster Architecture, Installation & Configuration - 25%
- [x] Manage role based access control (RBAC)
- [x] Use Kubeadm to install a basic cluster
- [x] Manage a highly-available Kubernetes cluster
- [ ] Provision underlying infrastructure to deploy a Kubernetes cluster
- [x] Perform a version upgrade on a Kubernetes cluster using Kubeadm
- [x] Implement etcd backup and restore

##### Services & Networking - 20%
- Understand host networking configuration on the cluster nodes
- Understand connectivity between Pods
- Understand ClusterIP, NodePort, LoadBalancer service types and endpoints
- Know how to use Ingress controllers and Ingress resources
- Know how to configure and use CoreDNS
- Choose an appropriate container network interface plugin

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

### Logs

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


