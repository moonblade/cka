

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



</details>

### Logs

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


