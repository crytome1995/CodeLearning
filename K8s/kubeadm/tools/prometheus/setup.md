ADD repos

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add kube-state-metrics https://kubernetes.github.io/kube-state-metrics
helm repo update

create namespace

kubectl create namespace monitoring

helm install prometheus prometheus-community/prometheus --namespace monitoring --values values.yaml

### NOTE

make sure storage class is configured first!!!!!
