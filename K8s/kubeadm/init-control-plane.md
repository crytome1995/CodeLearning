# Initialize the control plane of the master

1. Kubeadm init

```
kubeadm init
```

### Keep track of the kubeadm join command returned

init output:

```
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/
```

```
kubeadm join 10.0.3.171:6443 --token db0o82.kkvjvdr8fqu3chs3 \
        --discovery-token-ca-cert-hash sha256:5fe8caea37057abddd897f099b8c8efb45490e098433ba87344e8f8a
```

Apply a CNI to the cluster

```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
