Kind + ELK Kubernetes Setup Guide

Kubernetes Kind + Ingress + ELK Stack

Short, command-focused setup guide for Ubuntu 26.04.



1. Install Docker

sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker




2. Configure Docker Registry Mirrors

sudo bash -c 'cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": [
    "https://hub.hamdocker.ir"
  ]
}
EOF'

sudo systemctl restart docker




3. Install Kind

curl -Lo kind https://github.com/kubernetes-sigs/kind/releases/latest/download/kind-linux-amd64
chmod +x kind
sudo mv kind /usr/local/bin/




4. Create Kind Cluster (with containerd mirrors)

cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
    endpoint = ["https://hub.hamdocker.ir"]

  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.k8s.io"]
    endpoint = ["https://k8s-mirror.liara.ir"]

  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.elastic.co"]
    endpoint = ["https://elastic.hamdocker.ir"]

nodes:
  - role: control-plane
EOF


kind create cluster --config kind-config.yaml




5. Install Ingress-NGINX

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.0/deploy/static/provider/kind/deploy.yaml


kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --timeout=120s \
  -l app.kubernetes.io/component=controller




6. Create Logging Namespace

kubectl create namespace logging




7. Deploy Elasticsearch (Single Node)

kubectl apply -n logging -f https://raw.githubusercontent.com/elastic/elasticsearch/master/docs/examples/k8s/elasticsearch-single.yaml


Port forward:

kubectl port-forward svc/elasticsearch -n logging 9201:9200


Test:

curl http://localhost:9201




8. Deploy Logstash

ConfigMap (Beats input required)

kubectl apply -n logging -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-configmap
  namespace: logging
data:
  logstash.conf: |
    input {
      beats {
        port => 5000
      }
    }

    output {
      elasticsearch {
        hosts => ["http://elasticsearch.logging.svc.cluster.local:9200"]
        index => "nginx-logs-%{+YYYY.MM.dd}"
      }
      stdout { codec => rubydebug }
    }
EOF


Restart:

kubectl rollout restart deployment logstash -n logging




9. Deploy Filebeat (DaemonSet)

kubectl apply -n logging -f https://raw.githubusercontent.com/elastic/beats/master/deploy/kubernetes/filebeat-kubernetes.yaml




10. Test Logstash Connectivity

kubectl run test --rm -it --image=busybox --restart=Never -- \
  nc -zv logstash.logging.svc.cluster.local 5000


Expected:

open




11. Generate Logs (NGINX)

for i in {1..20}; do curl -s http://localhost > /dev/null; done




12. Verify Elasticsearch Index

curl http://localhost:9201/_cat/indices?v


Expected index:

nginx-logs-YYYY.MM.DD


View logs:

curl "http://localhost:9201/nginx-logs-*/_search?size=5&pretty"




Result





Kind cluster running



Registry mirrors configured



Ingress-NGINX working



Elasticsearch deployed



Logstash (Beats input) connected



Filebeat shipping container logs



Logs indexed in Elasticsearch



End of Guide
