### Create a managed k8s cluster

Managed clusters are far easier to set up and administrate and offer powerful auto-scaling capabilities.

1. set up the aws and eksctl cli tools locally

`aws configure`
[enter your access key id and secret)]

2. create a simple cluster (we will enable auto mode to make it as easy as possible)

`eksctl create cluster -f cluster.config`

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: [cluster name]
  region: eu-north-1

autoModeConfig:
  enabled: true
```

3. create the storage and ingress classes to enable basic load balancing and dynamic storage provision

```
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: eks.amazonaws.com/alb
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: auto-ebs-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.eks.amazonaws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"
```

### Installing from scratch on two EC2 nodes (Ubuntu 24.04)

Administering your own cluster gives you complete control, potentially improving extensibility, robustness, and reducing costs, but requires more administrative overhead.

was eksctl too easy for you?

lets build a cluster "hands-on", using the instructions from kubernetes.io, cilium, and the wider internet

At its most basic we need the kubelet, the api-server, the etcd, the kube-proxy, a CRI, and a CNI
    - most of these are covered by running kubeadm and/or the kubelet

many basic steps are left out for brevity's sake
    - e.g. adding apt repos, setting sysctl settings, swapoff, kernel module checks etc.

1. Install the kubelet, kubeadm and the CRI (we will use containerd. The addition of the corresponding debian repos has been left out):
    - `apt install kubelet=1.34.1-1.1 kubeadm=1.34.1-1.1 kubectl=1.34.1-1.1 containerd.io`

1. Configure containerd in `/etc/containerd/config.toml`: its important for us (since Ubuntu 24 uses systemd) to set the `SystemdCgroup` parameter to true
    - `containerd config default | sed /SystemdCgroup/s/false/true/g | tee /etc/containerd/config.toml`

1. Start/restart and enable the kubelet and containerd systemd services using `systemctl`

1. Start kubeadm. make sure to set the version and pod-network-cidr appropriately
    - `sudo kubeadm init --kubernetes-version=1.34.1 --pod-network-cidr=10.0.0.0/8`
    - make sure to look at the output and save the token so that you can run `kubeadm join` on any worker nodes you want to add

1. Now we just need a CNI plugin. We will go for cilium. Follow the download + install steps from their website:
    - `cilium install --version 1.18.3`
    - verify it worked with `cilium status --wait`

1. That should be all. Check everything is okay
    - `kubectl get nodes`
    - `kubectl get all -n kube-system`

1. Now we can add a worker node.
    - repeat all of the steps from above up until `kubeadm init` is run. then replace it with the `kubeadm join...` command you saved from the output of `kubeadm init`.
    - skip the cilium install step
    - perform the same checks as in the previous step

1. Troubleshooting:
    - make sure ip connectivity exists between the two nodes
        - `sysctl net.ipv4.ip_forward`
        - `ip r show`
        - `iptables-save`
        - vpc rules in aws
    - check logs: 
        - /var/log/kube-proxy.log
        - `ps -elf | grep kube-proxy`
        - `journalctl -u kubelet`
        - `kubectl -n kube-system logs kube-proxy-XXXX`

Excellent, now we are ready to deploy an app to your cluster!

### Installing the demo - openwebui + ollama

We will install openwebui - an open source UI for interacting with various LLM models, and complete with tool integrations and RAG capabilities.
As the backend, to host and run the LLM models we will use ollama.

- openwebui install is easy, like any generic webserver (e.g. nginx)
- ollama is not much different, however we would like to pre-install a model
    - initContainer vs lifecycleHook (PodStart) vs Probe vs building a custom image
        - I prefer an initContainer so that "ready" truly means ready
        - I saw a helm chart using the podStart
        - what if we want other probes later on...
        - image build is more work later if we want to change the model
- both require persistent storage
    - ollama for the models, openwebui for user data
- install both as a deployment with a corresponding service and a pvc
- add an ingress controller for routing external http traffic to openwebui
- livenessProbe had an initial delay that was too short, leading to restarts
    - increase its initialDelay
- we want to install a basic tool (e.g. weather) for the AI to use
    - no pre-built images -> alter the "command" for the python image, to fetch the git repo, install the pip packages, and run it
    - browser makes the requests not the container -> need another ingress 
- wow this is a lot of code and k8s declarations. lets use git to track it 
    ⁻ argocd integration - run a demo with the tool replica set

1. create a namespace and apply the ollama and webui manifests
```
kubectl create -n demo
kubectl apply -n demo -f ollama.yml
kubectl apply -n demo -f open-webui.yml
```

1. get our address from the loadbalancer
    - `kubectl -n demo get ingress open-webui-ingress -o jsonpath="{.items[0].status.loadBalancer.ingress[0].hostname}"`

1. and open it in the browser!

1. integrate argocd to be able to sync and store our manifests in git
    - `kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`
    - (optional) change the `type` of the argocd-server to `loadbalancer` to be able to access it on the nodeport

### Potential Improvements and Future Steps

- make backups of the etcd
- add more nodes to the cluster
- distribute etcd across multiple nodes 
- add GPUs
- create a helm chart
- add a storage class that maps in s3 buckets
- ssl certificates
- monitoring apps

