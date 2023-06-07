# Ping Pong API

## Container image
First we need to build the cotainer image:

```bash
cd ping-pong-api
docker build -t radu-ping-pong-api:0.1.1 .
```

## Creating a kubernetes cluster

### Option one, local development
For local development let's create a kubernetes cluster with a 
[kind](https://kind.sigs.k8s.io/) tool :) 

1. [Install `kind`](https://kind.sigs.k8s.io/#installation-and-usage)
```bash
go install sigs.k8s.io/kind@v0.19.0

kind --version
kind version 0.19.0
```

2. Create a kubernetes cluster

```bash
kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.27.1) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Thanks for using kind! 😊
```

3. Check that kubectl is connected to your cluster and you can run commands

```
kubectl config get-clusters
NAME
kind-kind

kubectl get ns
NAME                 STATUS   AGE
default              Active   2m22s
kube-node-lease      Active   2m22s
kube-public          Active   2m22s
kube-system          Active   2m22s
local-path-storage   Active   2m15s

kubectl get pods
No resources found in default namespace.
```

4. Add a ["LoadBalancer"](https://kind.sigs.k8s.io/docs/user/loadbalancer/)
service type.

Apply MetalLB manifest
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml

kubectl wait --namespace metallb-system \
                --for=condition=ready pod \
                --selector=app=metallb \
                --timeout=90s
```

Configure an IP range to use for your cluser LoaadBalancera.

Find out what network your local docker uses
```bash
docker network inspect -f '{{.IPAM.Config}}' kind
[{172.20.0.0/16  172.20.0.1 map[]} {fc00:f853:ccd:e793::/64  fc00:f853:ccd:e793::1 map[]}]
```

Create the resources below, make sure to check if the docker network matches.
If not replace `172.20.255.200` with that you get by running the cmd from
above.
```bash
kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example
  namespace: metallb-system
spec:
  addresses:
  - 172.20.255.200-172.20.255.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
EOF
```


### Option two, GKE cluster

To run commands in this section you need to 
[Install gcloud cli](https://cloud.google.com/sdk/docs/install)

GCP module based on based on https://github.com/terraform-google-modules/terraform-google-kubernetes-engine

Create the GCP project.
```bash
GCP_PRj=prj-radu-biconomy-prod-00001
gcloud projects create "${GCP_PRj}"

Create in progress for [https://cloudresourcemanager.googleapis.com/v1/projects/prj-radu-biconomy-prod-00001].
Waiting for [operations/cp.7492157465493557607] to finish...done.                                                                                                                                                                                             
Enabling service [cloudapis.googleapis.com] on project [prj-radu-biconomy-prod-00001]...
Operation "operations/acat.p2-858201302810-44a49c91-1c12-4388-a310-a88c381e2945" finished successfully.
```
[Running Terraform on your workstation.](https://registry.terraform.io/providers/hashicorp/google/latest/docs/guides/provider_reference#authentication)

```bash
gcloud auth application-default login
export TF_VAR_project_id=prj-radu-biconomy-prod-00001
```

Enable Compute Engine API 
``` bash
gcloud services enable compute.googleapis.com
```

Enable Kubernetes Engine API
```
gcloud services enablecontainer.googleapis.com
```

Run terraform commands

```
cd terraform-projects/prj-radu-biconomy-prod-00001
terraform init
terraform plan

terraform apply
```

In the end the output of `terraform apply` should look like this
```
module.gke.google_container_node_pool.pools["default-node-pool"]: Still creating... [50s elapsed]
module.gke.google_container_node_pool.pools["default-node-pool"]: Still creating... [1m0s elapsed]
module.gke.google_container_node_pool.pools["default-node-pool"]: Still creating... [1m10s elapsed]
module.gke.google_container_node_pool.pools["default-node-pool"]: Still creating... [1m20s elapsed]
module.gke.google_container_node_pool.pools["default-node-pool"]: Still creating... [1m30s elapsed]
module.gke.google_container_node_pool.pools["default-node-pool"]: Still creating... [1m40s elapsed]
module.gke.google_container_node_pool.pools["default-node-pool"]: Creation complete after 1m43s [id=projects/prj-radu-biconomy-prod-00001/locations/europe-west1/clusters/gke-on-vpc-cluster/nodePools/default-node-pool]

Apply complete! Resources: 9 added, 0 changed, 5 destroyed.

Outputs:

ca_certificate = <sensitive>
client_token = <sensitive>
cluster_name = "gke-on-vpc-cluster"
kubernetes_endpoint = <sensitive>
network_name = "gke-network"
service_account = "tf-gke-gke-on-vpc-clus-e3or@prj-radu-biconomy-prod-00001.iam.gserviceaccount.com"
subnet_name = [
  "gke-subnet",
]
subnet_secondary_ranges = [
  tolist([
    {
      "ip_cidr_range" = "192.168.0.0/18"
      "range_name" = "ip-range-pods"
    },
    {
      "ip_cidr_range" = "192.168.64.0/18"
      "range_name" = "ip-range-svc"
    },
  ]),
]

```


Check that yout cluster was created
```
gcloud container clusters list
NAME                LOCATION      MASTER_VERSION  MASTER_IP      MACHINE_TYPE  NODE_VERSION    NUM_NODES  STATUS
gke-on-vpc-cluster  europe-west1  1.25.8-gke.500  130.211.92.59  e2-medium     1.25.8-gke.500  3          RUNNING
```

## Running the application

### Option one, local development

1. Load the local conatiner image we build earlier `radu-ping-pong-api:0.1.1`
into the local kubernets cluster

```bash
kind load docker-image radu-ping-pong-api:0.1.1 
Image: "radu-ping-pong-api:0.1.1" with ID "sha256:1f8e2390d940f42d2775e2fe002ede570b7270edc3b929185a905aad03106214" not yet present on node "kind-control-plane", loading...
```
2. Create a deployment for the app

```bash
DEPLOYMENT=radu-ping-pong-api
kubectl create deployment "${DEPLOYMENT}" \
        --image=radu-ping-pong-api:0.1.1 \
				--replicas=2

deployment.apps/radu-ping-pong-api created

kubectl get deployment

NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
radu-ping-pong-api   1/1     1            1           42s

kubectl get pods
NAME                                 READY   STATUS    RESTARTS   AGE
radu-ping-pong-api-d9fcb99db-btrpm   1/1     Running   0          15s
radu-ping-pong-api-d9fcb99db-htopy   1/1     Running   0          15s

kubectl logs radu-ping-pong-api-d9fcb99db-btrpm

> ping-pong@1.0.0 start
> node server.js

Listening on PORT: 3000
```

3. Create a Kubernetes Service, which is a Kubernetes resource that lets us 
expose the application to external traffic.

```bash
PORT=80
DEPLOYMENT=radu-ping-pong-api
kubectl expose deployment "${DEPLOYMENT}" \
        --type=LoadBalancer \
        --port "${PORT}" \
        --target-port=3000

service/radu-ping-pong-api exposed

kubectl get svc
NAME                 TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
kubernetes           ClusterIP      10.96.0.1     <none>           443/TCP        6h18m
radu-ping-pong-api   LoadBalancer   10.96.33.70   172.20.255.200   80:31894/TCP   4s
```

4. Test the app
```bash
curl 172.20.255.200/ping
"pong"
```



### Option two, kubernetes cluster running on GKE

Install GKE auth plugin for gcloud
```bash
sudo apt-get install google-cloud-sdk-gke-gcloud-auth-plugin
```

Configure `kubectl` to connect to GKE cluster

```bash
gcloud container clusters get-credentials gke-on-vpc-cluster --region europe-west1

# check by running
kubectl get nodes
NAME                                                  STATUS   ROLES    AGE   VERSION
gke-gke-on-vpc-clust-default-node-poo-02c49e3c-xm5b   Ready    <none>   30m   v1.25.8-gke.500
gke-gke-on-vpc-clust-default-node-poo-232ba95e-m0rr   Ready    <none>   30m   v1.25.8-gke.500
gke-gke-on-vpc-clust-default-node-poo-85c02a9e-kqs2   Ready    <none>   30m   v1.25.8-gke.500
```

Push the docker image to the docker hub.
You would need an account on https://hub.docker.com/

```
pass="YOUR_DOCKER_HUB_PASSWORD"
user_docker="YOUR_DOCKER_HUB_USER"
docker login --username "${user_docker}" --password "${pass}"

docker image tag radu-ping-pong-api:0.1.1 "${user_docker}"/radu-ping-pong-api:0.1.1
$ e.g. docker image tag radu-ping-pong-api:0.1.1 radugp/radu-ping-pong-api:0.1.1

docker push "${user_docker}" /radu-ping-pong-api:0.1.1
# e.g. docker push radugp/radu-ping-pong-api:0.1.1 
```

Create the deployment
```bash
DEPLOYMENT=radu-ping-pong-api
kubectl create deployment "${DEPLOYMENT}" \
        --image=radugp/radu-ping-pong-api:0.1.1 \
				--replicas=2

kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
radu-ping-pong-api-598fbd6fbd-8f7bh   1/1     Running   0          2m50s
radu-ping-pong-api-598fbd6fbd-trlqb   1/1     Running   0          2m50s
```

Expose the deployment using a kubernetes service
```bash
PORT=80
DEPLOYMENT=radu-ping-pong-api
kubectl expose deployment "${DEPLOYMENT}" \
        --type=LoadBalancer \
        --port "${PORT}" \
        --target-port=3000


kubectl get svc
NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)        AGE
kubernetes           ClusterIP      192.168.64.1     <none>         443/TCP        56m
radu-ping-pong-api   LoadBalancer   192.168.77.233   34.22.168.53   80:30323/TCP   60s

curl 34.22.168.53/ping
"pong"
```


!!! Don't forget to run `terraform destroy` when done testing 
