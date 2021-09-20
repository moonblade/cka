

## Todo

- go thorugh kubernetes.io docs for better higher level overview understanding
- ~~lxc containers~~, mac only has lxc client
- [x] k8s the hard way
- [] RBAC


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


