
# Kubernetes Kind + Ingress + ELK Logging Stack

Short command‑focused guide for setting up a local Kubernetes lab using **Kind**, configuring **registry mirrors**, installing **Ingress‑NGINX**, and deploying a basic **ELK logging pipeline (Elasticsearch, Logstash, Filebeat)**.

Tested on: Ubuntu 26.04

---

# 1. Install Docker

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
```

---

# 2. Configure Docker Registry Mirror

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "registry-mirrors": [
    "https://hub.hamdocker.ir"
  ]
}
```

```bash
sudo systemctl restart docker
```

---

# 3. Install Kind

```bash
curl -Lo kind https://github.com/kubernetes-sigs/kind/releases/latest/download/kind-linux-amd64
chmod +x kind
sudo mv kind /usr/local/bin/
```

Verify:

```bash
kind --version
```

---

# 4. Create Kind Cluster with Registry Mirrors

```bash
nano [kind-config.yaml](.\Configs\kind-config.yaml)
```

```yaml
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
```

Create cluster:

```bash
kind create cluster --config kind-config.yaml
```

Check nodes:

```bash
kubectl get nodes
```

---

# 5. Install Ingress NGINX

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.0/deploy/static/provider/kind/deploy.yaml
```

Wait until ready:

```bash
kubectl wait --namespace ingress-nginx   --for=condition=ready pod   --selector=app.kubernetes.io/component=controller   --timeout=120s
```

---

# 6. Create Logging Namespace

```bash
kubectl create namespace logging
```

---

# 7. Deploy Elasticsearch (Single Node)

```bash
kubectl apply -n logging -f https://raw.githubusercontent.com/elastic/elasticsearch/master/docs/examples/k8s/elasticsearch-single.yaml
```

Check pods:

```bash
kubectl get pods -n logging
```

Port‑forward:

```bash
kubectl port-forward svc/elasticsearch -n logging 9200:9200
```

Test:

```bash
curl http://localhost:9200
```

---

# 8. Configure Logstash

Create ConfigMap:

```bash
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
```

Deploy Logstash:

```bash
kubectl apply -n logging -f https://raw.githubusercontent.com/elastic/logstash/master/docs/static/k8s/logstash-deployment.yaml
```

Restart if needed:

```bash
kubectl rollout restart deployment logstash -n logging
```

---

# 9. Deploy Filebeat (DaemonSet)

```bash
kubectl apply -n logging -f https://raw.githubusercontent.com/elastic/beats/master/deploy/kubernetes/filebeat-kubernetes.yaml
```

Check:

```bash
kubectl get pods -n logging
```

---

# 10. Test Logstash Connectivity

```bash
kubectl run test --rm -it --image=busybox --restart=Never -- nc -zv logstash.logging.svc.cluster.local 5000
```

Expected output:

```
open
```

---

# 11. Generate NGINX Logs

Send multiple HTTP requests:

```bash
for i in {1..20}; do curl http://localhost > /dev/null; done
```

---

# 12. Verify Elasticsearch Indices

```bash
curl http://localhost:9200/_cat/indices?v
```

Expected:

```
nginx-logs-YYYY.MM.DD
```

---

# 13. View Sample Logs

```bash
curl "http://localhost:9200/nginx-logs-*/_search?pretty&size=5"
```

---

# Result

Working pipeline:

```
NGINX → Filebeat → Logstash → Elasticsearch
```

Components deployed:

- Kind Kubernetes cluster
- Registry mirrors
- Ingress NGINX
- Elasticsearch
- Logstash (Beats input)
- Filebeat DaemonSet

