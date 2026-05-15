# Problems Faced During the Lab

## 1. Google Cloud Project Was Not Configured

### Issue

Initially, the active Google Cloud project was unset.

```text
project (unset)
PROJECT_ID=
```

### Cause

The Cloud Shell session started without an active project configuration.

### Solution

Configured the project manually using:

```bash
gcloud config set project qwiklabs-gcp-02-3e2774324577
```

### Result

The project ID was correctly configured and all further commands worked successfully.

---

## 2. Kubernetes Cluster Creation Took Time

### Issue

The GKE cluster creation process took several minutes.

### Observation

```text
Cluster is being health-checked (Kubernetes Control Plane is healthy)...
```

### Cause

GKE needs time to provision nodes, networking, and the Kubernetes control plane.

### Solution

Waited for the cluster provisioning process to complete successfully.

---

## 3. External LoadBalancer IP Was Not Immediately Available

### Issue

The frontend service initially did not return an external IP address.

### Cause

The LoadBalancer service required additional time for Google Cloud to provision an external IP.

### Solution

Repeated the following command after waiting:

```bash
kubectl get service frontend-external
```

or

```bash
echo "http://$(kubectl get service frontend-external -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')"
```

### Result

The external IP was assigned successfully.

---

## 4. Microservices Needed Time to Become Ready

### Issue

Some deployments were not immediately available after applying the Kubernetes manifests.

### Cause

Containers required time to pull images and start pods.

### Solution

Repeatedly checked deployment status:

```bash
kubectl get deployments
```

### Result

All microservices eventually reached:

```text
AVAILABLE = 1
```

---

## 5. Cloud Build Private Worker Pool Creation Failed

### Error

```text
ERROR: (gcloud.builds.worker-pools.create)
FAILED_PRECONDITION:
project "qwiklabs-gcp-02-3e2774324577" is unable to use private pools
```

### Cause

Private worker pools are disabled in the Qwiklabs lab environment.

### Solution

Followed the lab instructions and ignored the error because it was expected behavior in the temporary lab project.

---

## 6. Understanding Logs Explorer Queries

### Issue

It was initially difficult to write the correct Logs Explorer query for Kubernetes pod logs.

### Solution

Used Gemini assistance to generate the correct query:

```text
resource.type="k8s_container"
resource.labels.cluster_name="test"
resource.labels.namespace_name="default"
```

### Result

Successfully filtered and analyzed Kubernetes logs.

---

## 7. IAM Permission Propagation Delay

### Issue

After adding IAM roles, some services took a short time before permissions became active.

### Cause

Google Cloud IAM policy changes sometimes require propagation time.

### Solution

Waited briefly and retried the commands.

### Result

Permissions applied successfully and services became accessible.
