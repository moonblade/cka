

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
- Understand deployments and how to perform rolling update and rollbacks
- Use ConfigMaps and Secrets to configure applications
- Know how to scale applications
- Understand the primitives used to create robust, self-healing, application deployments
- Understand how resource limits can affect Pod scheduling
- Awareness of manifest management and common templating tools

##### Cluster Architecture, Installation & Configuration - 25%
- Manage role based access control (RBAC)
- Use Kubeadm to install a basic cluster
- Manage a highly-available Kubernetes cluster
- Provision underlying infrastructure to deploy a Kubernetes cluster
- Perform a version upgrade on a Kubernetes cluster using Kubeadm
- Implement etcd backup and restore

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


</details>


### Logs

> Day 4 - 22 Sep
- Check sample question for rbac, realize its easy to the point that you're overpreparing, and move onto kubeadm
- Created a new cluster with kubeadm, breezed through it, tried to create a multi master cluster, didn't have much luck due to not having a load balancer and couldn't figure out how to create one myself
So going to try multi cluster one with digital ocean
- Get your ass kicked by the humble kubeadm, wrangling an eel might have been easier. Anyway, docker is setup with cgroupfs instead of systemd, and kubelet doesn't start, whether or not its systemd and kubeadmin doesn't notice if it starts anyway

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


