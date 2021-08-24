# Join worker nodes to the cluster

Go to each worker node and run the init command returned from kubeadm init

```
kubeadm join 10.0.3.171:6443 --token 9p4zej.qwyyacgv9vzp5ejl \
        --discovery-token-ca-cert-hash sha256:3d240ef8f603d19021638c80cccc4909585b6f5aa2b4c2809b66ce1fc466b15f
```