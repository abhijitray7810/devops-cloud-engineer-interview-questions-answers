# Collect Metrics from Exporters using the Managed Service for Prometheus

## Objective

Deploy a GKE cluster with Managed Service for Prometheus, configure monitoring resources, run Prometheus and Node Exporter, and collect/export metrics successfully.

---

## Task 1: Deploy GKE Cluster

Create a GKE cluster with Managed Prometheus enabled:

```bash
gcloud beta container clusters create gmp-cluster \
--num-nodes=1 \
--zone us-east1-c \
--enable-managed-prometheus
```

Configure kubectl credentials:

```bash
gcloud container clusters get-credentials gmp-cluster --zone=us-east1-c
```

Verify cluster:

```bash
kubectl get nodes
```

---

## Task 2: Create Namespace

Create namespace for monitoring resources:

```bash
kubectl create ns gmp-test
```

Check GMP system pods:

```bash
kubectl get pods -n gmp-system
```

---

## Task 3: Deploy Example Application

Deploy sample application that exposes Prometheus metrics:

```bash
kubectl -n gmp-test apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.2.3/examples/example-app.yaml
```

Verify deployment:

```bash
kubectl get pods -n gmp-test
```

---

## Task 4: Configure PodMonitoring

Apply PodMonitoring resource to scrape metrics from the example application:

```bash
kubectl -n gmp-test apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.2.3/examples/pod-monitoring.yaml
```

Verify PodMonitoring resource:

```bash
kubectl get podmonitoring -A
```

---

## Task 5: Download Prometheus Binary

Clone Prometheus repository:

```bash
git clone https://github.com/GoogleCloudPlatform/prometheus
```

Enter repository:

```bash
cd prometheus
```

Checkout required version:

```bash
git checkout v2.28.1-gmp.4
```

Download Prometheus binary:

```bash
wget https://storage.googleapis.com/kochasoft/gsp1026/prometheus
```

Make executable:

```bash
chmod +x prometheus
```

---

## Task 6: Run Prometheus

Set project and zone variables:

```bash
export PROJECT=$(gcloud config get-value project)
export ZONE=us-east1-c
```

Run Prometheus:

```bash
./prometheus \
--config.file=documentation/examples/prometheus.yml \
--export.label.project-id=$PROJECT \
--export.label.location=$ZONE
```

---

## Task 7: Download and Run Node Exporter

Open another Cloud Shell tab.

Download Node Exporter:

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
```

Extract package:

```bash
tar xvfz node_exporter-1.3.1.linux-amd64.tar.gz
```

Enter directory:

```bash
cd node_exporter-1.3.1.linux-amd64
```

Run Node Exporter:

```bash
./node_exporter
```

Verify exporter running on port 9100.

---

## Create config.yaml

Create configuration file:

```bash
vi config.yaml
```

Add the following content:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: node
    static_configs:
      - targets: ['localhost:9100']
```

Save and exit.

---

## Upload Config File to GCS

Set project variable:

```bash
export PROJECT=$(gcloud config get-value project)
```

Create bucket:

```bash
gsutil mb -p $PROJECT gs://$PROJECT
```

Upload configuration:

```bash
gsutil cp config.yaml gs://$PROJECT
```

Make file public:

```bash
gsutil -m acl set -R -a public-read gs://$PROJECT
```

---

## Run Prometheus with New Config

Go back to prometheus directory:

```bash
cd ~/prometheus
```

Run Prometheus using config file:

```bash
./prometheus \
--config.file=/home/$USER/node_exporter-1.3.1.linux-amd64/config.yaml \
--export.label.project-id=$PROJECT \
--export.label.location=us-east1-c
```

---

## Verify Metrics

Open Web Preview on port 9090.

Run PromQL queries:

```promql
node_cpu_seconds_total
```

Other example queries:

```promql
node_memory_MemAvailable_bytes
```

```promql
up
```

Metrics and graphs should appear successfully.

---

## Result

Successfully deployed a GKE cluster with Managed Service for Prometheus, configured PodMonitoring, integrated Node Exporter, and visualized metrics in Prometheus UI.
