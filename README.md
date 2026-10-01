# SOFE 3980U Lab 3 Part 1 — Deploying on Google Kubernetes Engine

The Binary Calculator web application containerised and deployed to GKE,
together with the Kubernetes manifests that replace the imperative
`kubectl create` and `kubectl expose` commands used in the lab.

## Files

| Path | Description |
|---|---|
| `BinaryCalculatorWebapp/` | The Maven web application and its `Dockerfile` |
| `k8s/binarycalculator-deploy.yaml` | Deployment for the application (design part) |
| `k8s/binarycalculator-service.yaml` | LoadBalancer service for the application (design part) |
| `MySQL/mysql-deploy.yaml` | MySQL deployment used in the YAML section of the lab |
| `MySQL/mysql-service.yaml` | MySQL LoadBalancer service |

## Setting up the cluster

```
gcloud config set compute/zone us-central1-a

gcloud container clusters create sofe3980u \
  --num-nodes=3 --disk-size=50 --machine-type=e2-medium --zone=us-central1-a
```

## Deploying MySQL

Two techniques are shown in the lab. The imperative one:

```
kubectl create deployment mysql-deployment --image mysql/mysql-server --port=3306
kubectl expose deployment mysql-deployment --type=LoadBalancer --name=mysql-service
```

And the declarative one, which is preferable because the configuration is
versioned rather than living only in the cluster:

```
kubectl create -f MySQL/mysql-deploy.yaml
kubectl create -f MySQL/mysql-service.yaml
```

Connect once the external IP is assigned:

```
kubectl get service --watch
mysql -uuser -psofe3980u -h<IP-address>
```

## Building and deploying the application

Build the WAR, which the Dockerfile copies into the image:

```
cd BinaryCalculatorWebapp
mvn package
```

Create an Artifact Registry repository named `sofe3980u` in **us-central1
(Iowa)**, then grant the cluster permission to pull from it:

```
gcloud projects add-iam-policy-binding <PROJECT-ID> \
  --member="serviceAccount:$(gcloud projects describe <PROJECT-ID> \
      --format='value(projectNumber)')-compute@developer.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"
```

Build and push the image:

```
docker build -t <REPO-PATH>/binarycalculator .
docker push <REPO-PATH>/binarycalculator
```

`<REPO-PATH>` looks like
`us-central1-docker.pkg.dev/<PROJECT-ID>/sofe3980u`.

## Deploying with the YAML files (design part)

The lab deploys the application imperatively:

```
kubectl create deployment binarycalculator-deployment \
    --image <REPO-PATH>/binarycalculator --port=8080
kubectl expose deployment binarycalculator-deployment \
    --type=LoadBalancer --name=binarycalculator-service
```

The design task replaces both with manifests. First remove whatever is
already running:

```
kubectl delete deployment binarycalculator-deployment
kubectl delete service binarycalculator-service
```

Set the image path in `k8s/binarycalculator-deploy.yaml` (replace
`REPLACE_WITH_REPO_PATH`), then apply both files:

```
kubectl apply -f k8s/binarycalculator-deploy.yaml
kubectl apply -f k8s/binarycalculator-service.yaml
```

Get the external IP and open the application:

```
kubectl get service binarycalculator-service --watch
```

```
http://<EXTERNAL-IP>:8080
```

## What the manifests add

Beyond replacing the two commands, the deployment manifest declares things the
imperative form cannot express:

- a **readiness probe**, so traffic is only sent once the application responds
- a **liveness probe**, so a container that stops responding is restarted
- **resource requests and limits**, so the scheduler knows what the pod needs

The service is also kept separate from the deployment. A LoadBalancer service
holds its external IP for as long as it exists, so the deployment can be
replaced without the URL changing.

## Cleaning up

A running cluster consumes credit continuously.

```
kubectl delete -f k8s/binarycalculator-service.yaml
kubectl delete -f k8s/binarycalculator-deploy.yaml
kubectl delete -f MySQL/mysql-service.yaml
kubectl delete -f MySQL/mysql-deploy.yaml

gcloud container clusters delete sofe3980u --zone=us-central1-a
```
