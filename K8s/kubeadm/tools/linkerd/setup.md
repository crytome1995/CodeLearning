https://linkerd.io/2.10/getting-started/

install linkerd cli

curl -sL linkerd https://github.com/linkerd/linkerd2/releases/download/stable-2.10.2/linkerd2-cli-stable-2.10.2-linux-amd64 \
 && chmod +x linkerd
&& mv ./linkerd /usr/local/bin/

validate cluster

linkerd check --pre

install control plane

linkerd install | kubectl apply -f -

setup dashboards

linkerd viz install --set prometheusUrl="http://prometheus-server.monitoring.svc.cluster.local",prometheus.enabled=false | kubectl apply -f -

label default namespace

kubectl annotate namespace default linkerd.io/inject=enabled
