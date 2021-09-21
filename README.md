

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

### Logs

> Day 3 - 21 Sep
- Reading about users, setup some rbac basic roles and bindings,
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


