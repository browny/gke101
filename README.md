// @format

# GKE 101

## Lab1

Introduction to Containers and Docker

### Run the web server manually

Python

```bash
# Provision a debian VM
gcloud compute instances create webserver \
  --zone=asia-east1-b --machine-type=g1-small --image-family=debian-9 --image-project=debian-cloud

# Run web-server as VM-based style (SSH into VM `webserver`)
sudo apt-get update
sudo apt-get install -y python3 python3-pip
pip3 install tornado
python3 web-server.py &

curl http://localhost:8888

# Create firewall rule to open internet access

# Terminate
kill %1
```

Nodejs

```bash
# Provision a debian VM
gcloud compute instances create webserver \
  --zone=asia-east1-b --machine-type=g1-small --image-family=debian-9 --image-project=debian-cloud

# Config Nodejs env (SSH into VM `webserver`)
sudo apt-get install curl software-properties-common
curl -sL https://deb.nodesource.com/setup_12.x | sudo bash -
sudo apt-get install -y nodejs

# Code a simple web service
npm init
npm install express --save
vim hello.js

---
var express = require('express');
var app = express();

app.get('/', function (req, res) {
   res.send('Hello World from GCE!');
});

app.listen(3000, function () {
   console.log('Example app listening on port 3000!');
});
---

node hello.js &
curl http://localhost:3000

# Create firewall rule to open internet access

# Terminate
kill %1
```

### Package using Docker

Python

```bash
# Cloud Shell
git clone https://github.com/browny/gke101.git
cd gke101/lab1

# Upload Dockerfile
cat Dockerfile

# Build container image
docker build -t py-web-server:v1 .

# Run container
docker run -d -p 8888:8888 --name py-web-server -h my-web-server py-web-server:v1

curl http://localhost:8888
docker rm -f py-web-server
```

Nodejs

```bash
# Cloud Shell
git clone https://github.com/browny/gke101.git
cd gke101/lab1

# Upload Dockerfile
cat Dockerfile-nodejs

# Build container image
docker build -t node-web-server:v1 -f Dockerfile-nodejs .

# Run container
docker run -d -p 3000:3000 --name node-web-server -h my-web-server node-web-server:v1

curl http://localhost:3000
docker rm -f node-web-server
```

### Upload the image to a registry

Python

```bash
export GCP_PROJECT=`gcloud config list core/project --format='value(core.project)'`

# Rebuild the Docker image with a registry name that includes gcr.io as the hostname and the project
# ID as a prefix
docker build -t "gcr.io/${GCP_PROJECT}/py-web-server:v1" .

# Configure Docker to use gcloud as a Container Registry credential helper (you are only required to
# do this once).
gcloud auth configure-docker

# Push image to Container Registry
docker push gcr.io/${GCP_PROJECT}/py-web-server:v1

gcloud container images list-tags gcr.io/${GCP_PROJECT}/py-web-server

# Make image public accessible (optional), then you can run anywhere
gsutil iam ch allUsers:objectViewer "gs://artifacts.${GCP_PROJECT}.appspot.com"
docker run -d -p 8080:8888 -h my-web-server gcr.io/${GCP_PROJECT}/py-web-server:v1
```

Nodejs

```bash
export GCP_PROJECT=`gcloud config list core/project --format='value(core.project)'`

# Rebuild the Docker image with a registry name that includes gcr.io as the hostname and the project
# ID as a prefix
docker build -t "gcr.io/${GCP_PROJECT}/node-web-server:v1" -f Dockerfile-nodejs .

# Configure Docker to use gcloud as a Container Registry credential helper (you are only required to
# do this once).
gcloud auth configure-docker

# Push image to Container Registry
docker push gcr.io/${GCP_PROJECT}/node-web-server:v1

gcloud container images list-tags gcr.io/${GCP_PROJECT}/node-web-server

# Make image public accessible (optional), then you can run anywhere
gsutil iam ch allUsers:objectViewer "gs://artifacts.${GCP_PROJECT}.appspot.com"
docker run -d -p 8080:3000 -h my-web-server gcr.io/${GCP_PROJECT}/node-web-server:v1
```

### Run the web server container on Compute Engine

Python

```bash
# Run container on Compute Engine instance
gcloud beta compute instances create-with-container py-web-server --zone=asia-east1-b \
  --machine-type=g1-small --tags=web-server \
  --container-image="gcr.io/${GCP_PROJECT}/py-web-server:v1"

# Expose to internet
gcloud compute firewall-rules create allow-8888 --direction=INGRESS \
  --priority=1000 --network=default --action=ALLOW --rules=tcp:8888 --source-ranges=0.0.0.0/0 \
  --target-tags=web-server
```

Nodejs

```bash
# Run container on Compute Engine instance
gcloud beta compute instances create-with-container node-web-server --zone=asia-east1-b \
  --machine-type=g1-small --tags=web-server \
  --container-image="gcr.io/${GCP_PROJECT}/node-web-server:v1"

# Expose to internet
gcloud compute firewall-rules create allow-3000 --direction=INGRESS \
  --priority=1000 --network=default --action=ALLOW --rules=tcp:3000 --source-ranges=0.0.0.0/0 \
  --target-tags=web-server

```

## Lab2

Kubernetes Basics

### Start a kubernetes cluster

```bash
# Press `Connect` button to configure kubectl command
Run in Cloud Shell

# (optional) make kubectl with auto-completion
source <(kubectl completion bash)
cd gke101/lab2
```

### Run and deploy a container

```bash
# kubectl run nginx --image=nginx:1.10.0 --generator=deployment/apps.v1beta1 --dry-run -o yaml
kubectl create -f deploy.yaml

kubectl get pods
kubectl get pods -o wide
```

### Expose service

```bash
kubectl expose deployment nginx --port 80 --type LoadBalancer
kubectl get services
```

### Scale up

````bash
kubectl scale deployment nginx --replicas 3
kubectl get pods
kubectl get services # external IP has not changed

curl http://<External IP>:80


### Load testing

```bash
gcloud compute instances create loadtest \
  --zone=asia-east1-b --machine-type=g1-small --image-family=debian-9 --image-project=debian-cloud

sudo apt-get update
sudo apt-get install -y wrk
wrk -t4 -c100 -d30s http://<External IP>:80

# https://github.com/wercker/stern (Multi pod and container log tailing for Kubernetes)
# https://github.com/wercker/stern/releases, upload stern_linux_amd64
sudo mv stern_linux_amd64 /usr/local/bin/stern
sudo chmod 777 /usr/local/bin/stern
stern "nginx.*"
````

### Clean up

```bash
kubectl delete deployment nginx
kubectl delete service nginx
```

## Lab3

https://github.com/googlecodelabs/orchestrate-with-kubernetes

```bash
# Get sample codes
git clone https://github.com/googlecodelabs/orchestrate-with-kubernetes.git
cd orchestrate-with-kubernetes/kubernetes
```

### Pods

```bash
# Deploy pods
kubectl explain pods
cat pods/monolith.yaml

kubectl create -f pods/monolith.yaml
kubectl describe pods monolith
```

### Interacting with pods

```bash
# port forwarding (keep terminal running)
kubectl port-forward monolith 10080:80
curl http://127.0.0.1:10080

# fail bcz you need to include an auth token in your request
curl http://127.0.0.1:10080/secure

# login to get token
TOKEN=$(curl http://127.0.0.1:10080/login -u user|jq -r '.token')
password: `password`

curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:10080/secure

# logging
kubectl logs -f monolith

# login into container
kubectl exec monolith --stdin --tty -c monolith /bin/sh

# test internet connectivity inside container
ping -c 3 google.com
```

### Monitoring and health checks

#### Readiness (Readiness probes indicate when a pod is "ready" to serve traffic)

```bash
cat pods/healthy-monolith.yaml

kubectl create -f pods/healthy-monolith.yaml
kubectl describe pod healthy-monolith

kubectl port-forward healthy-monolith 10081:81
# force the monolith container readiness probe to fail (toggle the readiness probe status)
curl http://127.0.0.1:10081/readiness/status

# Check READY -> 0/1
kubectl get pods healthy-monolith -w

# Readiness probe failed: HTTP probe failed with statuscode: 503
kubectl describe pods healthy-monolith
```

#### Liveness (Liveness probes indicate whether a container is "alive.")

```bash
kubectl port-forward healthy-monolith 10081:81

curl http://127.0.0.1:10081/healthz/status

# Wait for pod restart
kubectl get pods healthy-monolith -w

kubectl describe pods healthy-monolith
```

### Services

```bash
# Create secret and configmap
kubectl create secret generic tls-certs --from-file tls/
kubectl create configmap nginx-proxy-conf --from-file nginx/proxy.conf

cat nginx/proxy.conf

# Create pods
cat pods/secure-monolith.yaml
kubectl create -f pods/secure-monolith.yaml

# Create services
cat services/monolith.yaml
kubectl create -f services/monolith.yaml

# Create firewall for external access
gcloud compute firewall-rules create allow-monolith-nodeport --allow=tcp:31000

# Why not work (labels)
gcloud compute instances list | grep gke-
https://<EXTERNAL_IP>:31000

# Add labels to pods
kubectl get pods -l "app=monolith,secure=enabled" # nothing
kubectl label pods secure-monolith 'secure=enabled'
kubectl get pods secure-monolith --show-labels

# Try again
gcloud compute instances list | grep gke-
open https://<EXTERNAL_IP>:31000
```
