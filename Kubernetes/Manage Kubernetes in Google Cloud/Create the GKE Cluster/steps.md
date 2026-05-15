## Task 1 — Create the GKE Cluster

Set variables first:

```bash
export ZONE=YOUR_ZONE
export CLUSTER_NAME=YOUR_CLUSTER_NAME
export NAMESPACE=YOUR_NAMESPACE
export REPO_NAME=YOUR_REPO_NAME
export SERVICE_NAME=YOUR_SERVICE_NAME
```

Enable required APIs:

```bash
gcloud services enable \
container.googleapis.com \
monitoring.googleapis.com \
artifactregistry.googleapis.com
```

Create cluster:

```bash
gcloud container clusters create $CLUSTER_NAME \
  --zone=$ZONE \
  --release-channel=regular \
  --num-nodes=3 \
  --enable-autoscaling \
  --min-nodes=2 \
  --max-nodes=6
```

Connect to cluster:

```bash
gcloud container clusters get-credentials $CLUSTER_NAME --zone=$ZONE
```

---

## Task 2 — Enable Managed Prometheus

Enable managed collection:

```bash
gcloud container clusters update $CLUSTER_NAME \
  --zone=$ZONE \
  --enable-managed-prometheus
```

Create namespace:

```bash
kubectl create namespace $NAMESPACE
```

Download manifest:

```bash
gcloud storage cp gs://spls/gsp510/prometheus-app.yaml .
```

Edit file:

```bash
nano prometheus-app.yaml
```

Replace `<todo>` values with:

```yaml
image: nilebox/prometheus-example-app:latest
name: prometheus-test
ports:
  - name: metrics
```

Apply manifest:

```bash
kubectl apply -f prometheus-app.yaml -n $NAMESPACE
```

Download monitoring file:

```bash
gcloud storage cp gs://spls/gsp510/pod-monitoring.yaml .
```

Edit:

```bash
nano pod-monitoring.yaml
```

Replace `<todo>` with:

```yaml
metadata:
  name: prometheus-test

labels:
  app.kubernetes.io/name: prometheus-test

matchLabels:
  app: prometheus-test

interval: 30s
```

Apply:

```bash
kubectl apply -f pod-monitoring.yaml -n $NAMESPACE
```

---

## Task 3 — Deploy Broken Application

Download app:

```bash
gcloud storage cp -r gs://spls/gsp510/hello-app/ .
```

Deploy manifest:

```bash
kubectl apply -f hello-app/manifests/helloweb-deployment.yaml -n $NAMESPACE
```

Check deployment:

```bash
kubectl get pods -n $NAMESPACE
kubectl describe pod POD_NAME -n $NAMESPACE
```

You should see:

```text
InvalidImageName
```

---

## Task 4 — Create Logs-based Metric and Alert

Go to:

```text
Logging → Logs Explorer
```

Use query:

```text
resource.type="k8s_pod"
severity>=WARNING
```

Run query and verify errors appear.

Create logs metric:

* Metric Type: Counter
* Name: `pod-image-errors`

Then go to:

```text
Monitoring → Alerting → Create Policy
```

Use:

* Metric: `pod-image-errors`
* Rolling Window: `10 min`
* Function: `Count`
* Aggregation: `Sum`
* Condition: `Above threshold`
* Threshold: `0`
* Trigger: `Any time series violates`
* Notification channel: none
* Policy name: `Pod Error Alert`

---

## Task 5 — Fix Deployment

Edit deployment file:

```bash
nano hello-app/manifests/helloweb-deployment.yaml
```

Replace image with:

```yaml
image: us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0
```

Delete broken deployment:

```bash
kubectl delete deployment helloweb -n $NAMESPACE
```

Redeploy:

```bash
kubectl apply -f hello-app/manifests/helloweb-deployment.yaml -n $NAMESPACE
```

Verify:

```bash
kubectl get pods -n $NAMESPACE
```

Pods should become `Running`.

---

## Task 6 — Build, Push, and Deploy v2 Image

Edit Go app:

```bash
nano hello-app/main.go
```

Change:

```go
Version: 2.0.0
```

Configure Docker auth:

```bash
gcloud auth configure-docker
```

Get project ID:

```bash
export PROJECT_ID=$(gcloud config get-value project)
```

Build image:

```bash
cd hello-app
```

```bash
docker build -t us-docker.pkg.dev/$PROJECT_ID/$REPO_NAME/hello-app:v2 .
```

Push image:

```bash
docker push us-docker.pkg.dev/$PROJECT_ID/$REPO_NAME/hello-app:v2
```

Update deployment image:

```bash
kubectl set image deployment/helloweb \
  hello-app=us-docker.pkg.dev/$PROJECT_ID/$REPO_NAME/hello-app:v2 \
  -n $NAMESPACE
```

Expose deployment:

```bash
kubectl expose deployment helloweb \
  --name=$SERVICE_NAME \
  --type=LoadBalancer \
  --port=8080 \
  --target-port=8080 \
  -n $NAMESPACE
```

Get external IP:

```bash
kubectl get svc -n $NAMESPACE
```

Open external IP in browser:

Expected output:

```text
Hello, world!
Version: 2.0.0
```

Useful verification commands:

```bash
kubectl get all -n $NAMESPACE
```

```bash
kubectl logs deployment/helloweb -n $NAMESPACE
```

```bash
kubectl describe svc $SERVICE_NAME -n $NAMESPACE
```
