# Manage Kubernetes in Google Cloud

I did the following:

1. Created a GKE cluster based on a set of configurations provided.
2. Enabled Managed Prometheus on the cluster for metrics monitoring.
3. Deployed a Kubernetes manifest onto the cluster, and debugged the errors.
4. Created a logs-based metric and alerting policy for the errors on the Kubernetes cluster.
5. Fixed the manifest errors, containerized the application code and pushed it to Artifact Registry using Docker.
6. Exposed a service for the application on the cluster and verified the updates.

Remember the following important Kubernetes concepts:

#### Container Image: A lightweight, standalone, executable package that includes everything needed to run a piece of software (code, runtime, libraries).

#### Pod: The smallest deployable unit in Kubernetes, hosting one or more tightly coupled containers that share the same network and storage.

#### Node: A worker machine (either physical or virtual) in Kubernetes that runs your pods and is managed by the master control plane.

#### Deployment: A declarative blueprint that automates how many copies of a pod should run, handles updates, and ensures self-healing if they fail.

#### Service: An abstract way to expose a logical set of pods as a network service, giving them a single, permanent IP address and load-balancing traffic.

#### Cluster: A complete, interconnected set of nodes grouped together to run your containerized applications as a single unified system

---

## Set the Environmental Variable

```bash
export REGION=us-east1
export ZONE=us-east1-d
```

---

## Task 1: Creating a GKE Cluster Based on a Set of Configurations Provided

| Setting | Value |
|---|---|
| Zone | ZONE |
| Release channel | Regular |
| Cluster/Target version | default |
| Cluster autoscaler | Enabled |
| Number of nodes | 3 |
| Minimum nodes | 2 |
| Maximum nodes | 6 |

```bash
gcloud container clusters create hello-world-38je \
    --zone=$ZONE \
    --release-channel=regular \
    --num-nodes=3 \
    --enable-autoscaling \
    --min-nodes=2 \
    --max-nodes=6
```

---

## Task 2: Enabling Managed Prometheus on the Cluster for Metrics Monitoring

**1. Enable the Prometheus managed collection on the GKE cluster.**

```bash
gcloud container clusters update hello-world-38je --zone=$ZONE --enable-managed-prometheus

gcloud container clusters get-credentials hello-world-38je --zone=$ZONE
```

**2. Create a namespace on the cluster named `<namespace name>`.**

```bash
kubectl create ns gmp-ad6o
```

**3. Download a sample Prometheus app:**

```bash
gcloud storage cp gs://spls/gsp510/prometheus-app.yaml .
```

**4. Update the `<todo>` sections (lines 35-38) with the following configuration:**

- `containers.image`: `nilebox/prometheus-example-app:latest`
- `containers.name`: `prometheus-test`
- `ports.name`: `metrics`

**5. Deploy the application onto the `<namespace name>` namespace on your GKE cluster.**

```bash
kubectl apply -f prometheus-app.yaml -n gmp-ad6o
```

**6. Download the `pod-monitoring.yaml` file:**

```bash
gcloud storage cp gs://spls/gsp510/pod-monitoring.yaml .
```

**7. Update the `<todo>` sections (lines 18-24) with the following configuration:**

- `metadata.name`: `prometheus-test`
- `labels.app.kubernetes.io/name`: `prometheus-test`
- `matchLabels.app`: `prometheus-test`
- `endpoints.interval`: interval period

**8. Apply the pod monitoring resource onto the `<namespace name>` namespace on your GKE cluster.**

```bash
kubectl apply -f pod-monitoring.yaml -n gmp-ad6o
```

**Verify:**

```bash
kubectl get podmonitoring -n gmp-ad6o
```

---

## Task 3: Deploy an Application onto the GKE Cluster

Deploying a Kubernetes manifest onto the cluster, and debugging the errors.

**1. Download the demo deployment manifest files:**

```bash
gcloud storage cp -r gs://spls/gsp510/hello-app/ .
```

**2. Create a deployment onto the `<namespace name>` namespace on your GKE cluster from the `helloweb-deployment.yaml` manifest file. It is located in the `hello-app/manifests` folder.**

```bash
kubectl -n gmp-ad6o create -f hello-app/manifests/helloweb-deployment.yaml
```

Example:
```bash
kubectl -n <Namespace> create -f helloweb-deployment.yaml
```

**3. Verify you have created the deployment, and navigate to the helloweb deployment details page. You should see the following error:**

```bash
kubectl get deployments helloweb --namespace=gmp-ad6o
```

---

## Task 4: Creating a Logs-Based Metric and Alerting Policy for the Errors on the Kubernetes Cluster

**1. In the Logs Explorer, create a query that exposes warnings/errors you saw in the previous section on the cluster.**

> Console > Navigation menu > Logging > Logs Explorer > Enable **Show query** > In the Query builder box, add the following query:

```
resource.type="k8s_pod"
severity=WARNING
```

### Explanation: This query searches your logs for Kubernetes pods that are experiencing warning-level issues or errors. 

Click **Run Query**.

**Create the logs-based metric:**

Click **Actions** dropdown > Select **Create Metric** > Name: `pod-image-errors` > **Create Metric** to save the log-based metric.

Or use the CLI command:

```bash
gcloud logging metrics create pod-image-errors \
    --description="Counts image-related errors for Kubernetes pods" \
    --log-filter='resource.type="k8s_pod" AND severity="WARNING"'
```

To delete a metric named `pod-image-errors`:

```bash
gcloud logging metrics delete pod-image-errors --quiet
```

**2. Create a logs-based metric from this query. For Metric type, use `Counter` and for the Log Metric Name use `pod-image-errors`.**

Create an Alerting Policy based on the logs-based metric you just created. Use the following details to configure your policy:

> Console > Monitoring > Alerting > Create policy > Select a metric: Filter by the name of your metric: `pod-image-errors` > Click **Kubernetes Pod** > **Log-based metrics** > **Logging/user/pod-image-errors** > **Apply** > Then follow the configurations below:

| Setting | Value |
|---|---|
| Rolling Window | 10 min |
| Rolling window function | Count |
| Time series aggregation | Sum |
| Condition type | Threshold |
| Alert trigger | Any time series violates |
| Threshold position | Above threshold |
| Threshold value | 0 |
| Use notification channel | Disable |
| Alert policy name | Pod Error Alert |

To delete an alert named `Pod Error Alert`:

```bash
POLICY_ID=$(gcloud alpha monitoring policies list --filter="displayName='Pod Error Alert'" --format="value(name)")

if [ -z "$POLICY_ID" ]; then
    echo "Alert policy 'Pod Error Alert' not found."
else
    gcloud alpha monitoring policies delete "$POLICY_ID" --quiet
    echo "Successfully deleted policy."
fi
```

---

## Task 5: Update and Re-Deploy Your App

Fixing the manifest errors, containerizing your application code and pushing it to Artifact Registry using Docker.

The development team would like to see you demonstrate your knowledge on deleting and updating deployments on the cluster in case of an error. In this section, you will update a Kubernetes manifest with a correct image reference, delete a deployment, and deploy the updated application onto the cluster.

**1. Replace the `<todo>` in the image section in the `helloweb-deployment.yaml` deployment manifest with the following image:**

```
us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0
```

**2. Delete the helloweb deployment from your cluster.**

```bash
kubectl delete deployment helloweb --namespace=gmp-ad6o
```

Example:
```bash
kubectl delete deployment <name_of_deployment> --namespace=<namespace>
```

**3. Deploy the updated `helloweb-deployment.yaml` manifest onto your cluster on the `<namespace name>` namespace.**

```bash
kubectl -n gmp-ad6o create -f hello-app/manifests/helloweb-deployment.yaml
```

---

## Task 6: Containerize Your Application Code, Push It to Artifact Registry Using Docker and Deploy It onto the Cluster

Expose a service for your application on the cluster and verify your updates.

**1. In the `hello-app` directory, update the `main.go` file to use `Version: 2.0.0` on line 49.**

**2. Use the `hello-app/Dockerfile` to create a Docker image with the `v2` tag.**

```bash
docker build -t us-east1-docker.pkg.dev/qwiklabs-gcp-02-35a03730c6e1/hello-repo/hello-app:v2 .
```

Example:
```bash
docker build -t REGION-docker.pkg.dev/PROJECT_ID/REPOSITORY_NAME/IMAGE_NAME:v2 .
```

**3. Push the newly built Docker image to your repository in Artifact Registry using the `v2` tag.**

First, ensure Docker is authenticated to interact with your Google Cloud Artifact Registry region:

```bash
gcloud auth configure-docker us-east1-docker.pkg.dev

docker push us-east1-docker.pkg.dev/qwiklabs-gcp-02-35a03730c6e1/hello-repo/hello-app:v2
```

**4. Set the image on your helloweb deployment to reflect the `v2` image you pushed to Artifact Registry.**

```bash
kubectl set image deployment/helloweb hello-app=us-east1-docker.pkg.dev/qwiklabs-gcp-02-35a03730c6e1/hello-repo/hello-app:v2 -n gmp-ad6o
```

Example:
```bash
kubectl set image deployment/helloweb hello-app=LOCATION-docker.pkg.dev/PROJECT_ID/REPOSITORY_NAME/hello-app:v2 --namespace=gmp-uqkw
```

**5. Expose the helloweb deployment to a LoadBalancer service named `<service name>` on port 8080, and set the target port of the container to the one specified in the Dockerfile.**

```bash
kubectl expose deployment helloweb \
    --name=helloweb-service-69ku \
    --type=LoadBalancer \
    --port=8080 \
    --target-port=8080 \
    --namespace=gmp-ad6o
```

Example:
```bash
kubectl expose deployment helloweb \
    --name=<service name> \
    --type=LoadBalancer \
    --port=8080 \
    --target-port=8080 \
    --namespace=<namespace>
```

**6. Navigate to the external load balancer IP address of the service `<name service>`, and you should see the following text returned by the service:**

```bash
curl http://34.91.48.99

kubectl get service helloweb-service-qvzg --namespace=gmp-yqnz -o wide
```

Example:
```bash
kubectl get service <service name> -n namespace
```

---

## Notes

- **Service resource (LoadBalancer)** — the object through which your containerized app sitting in pods can reach the internet.
- **Deployment Object** — creates, rolls out, replicaSet — ensures desired number of pods (specified in the deployment manifest) are running.

## Recommendation: To understand applicable Kubernetes more, do this Lab on Google Skills: Debug Apps on Google Kubernetes Engine. 
https://www.skills.google/focuses/13065?parent=catalog
My best lab so far. So applicable to real-world application systems and detailed.  
