This is a kubeadm created kubernetes playground wrapped in a Vagrant environment.
* Ubuntu 20.04 Focal Fossa
* Kubernetes 1.20
* Docker 19.03

# Usage
Launch a kubernetes:

```bash
vagrant up

export KUBECONFIG=$(pwd)/tmp/admin.conf

kubectl cluster-info
kubectl get node
kubectl get pod --all-namespaces
```
