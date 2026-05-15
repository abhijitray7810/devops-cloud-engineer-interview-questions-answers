
````markdown
# Managed Service for Prometheus on GKE

## Objective
Set up Google Kubernetes Engine (GKE) with Managed Service for Prometheus and verify metrics collection using Node Exporter.

---

# Step 1: Verify Organization Policy

```bash
gcloud org-policies describe constraints/gcp.resourceLocations \
--effective --project=$(gcloud config get-value project)
````

---

# Step 2: Create GKE Cluster with Managed Prometheus

```bash
gcloud beta container clusters create gmp-cluster \
--num-nodes=1 \
--zone=us-west1-b \
--enable-managed-prometheus
```

---

# Step 3: Configure kubectl Access

```bash
gcloud container clusters get-credentials gmp-cluster --zone=us-west1-b
```

---

# Step 4: Create Namespace

```bash
kubectl create ns gmp-test
```

---

# Step 5: Clone Prometheus Repository

```bash
git clone https://github.com/GoogleCloudPlatform/prometheus
cd prometheus
```

Checkout required version:

```bash
git checkout v2.28.1-gmp.4
```

---

# Step 6: Download Prometheus Binary

```bash
wget https://storage.googleapis.com/kochasoft/gsp1026/prometheus
chmod a+x prometheus
```

---

# Step 7: Set Environment Variables

```bash
export PROJECT=$(gcloud config get-value project)
export ZONE=us-west1-b
```

---

# Step 8: Download Node Exporter

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
```

Extract files:

```bash
tar xvfz node_exporter-1.3.1.linux-amd64.tar.gz
```

Move into directory:

```bash
cd node_exporter-1.3.1.linux-amd64
```

---

# Step 9: Start Node Exporter

```bash
./node_exporter
```

Expected Output:

```text
Listening on address=:9100
```

---

# Step 10: Create Prometheus Configuration

Create `config.yaml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: node
    static_configs:
      - targets: ['localhost:9100']
```

---

# Step 11: Create Cloud Storage Bucket

```bash
gsutil mb -p $PROJECT gs://$PROJECT
```

Upload config file:

```bash
gsutil cp config.yaml gs://$PROJECT
```

Make bucket publicly readable:

```bash
gsutil -m acl set -R -a public-read gs://$PROJECT
```

---

# Step 12: Start Prometheus

```bash
cd ~/prometheus

./prometheus \
--config.file=config.yaml \
--export.label.project-id=$PROJECT \
--export.label.location=$ZONE
```

Expected Output:

```text
Server is ready to receive web requests.
```

---

# Step 13: Verify Prometheus Metrics

Check Prometheus metrics:

```bash
curl localhost:9090/metrics | head
```

Check Node Exporter metrics:

```bash
curl localhost:9100/metrics | head
```

---

# Step 14: Verify Prometheus Targets

```bash
curl localhost:9090/api/v1/targets
```

Expected Result:

```json
"health":"up"
```

---

# Step 15: Verify GKE Managed Prometheus Pods

```bash
kubectl get pods -n gmp-system
```

Expected Running Pods:

* collector
* gmp-operator

---

# Step 16: Verify All Kubernetes Pods

```bash
kubectl get pods -A
```

---

# Result

Successfully configured:

* Google Kubernetes Engine (GKE)
* Managed Service for Prometheus
* Node Exporter
* Prometheus metrics scraping
* Cloud Monitoring integration

Metrics endpoint status verified as:

```json
"health":"up"
```

```
```
