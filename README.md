# distributed-nats-akamaicompute

This is a cookbook for deploying a meshed, distributed NATS.io cluster across multiple Akamai Compute Regions via Linode Kubernetes Engine. 

![image](https://user-images.githubusercontent.com/19197357/225468591-5e601e39-3e74-4437-a175-4645a261be87.png)

NATS.io is a modern pub/sub protocol that can fulfill both legacy, MQTT-based use cases, as well as modern messaging applications that rely on NATS.io, or connectors to event-based data and application platforms such as HarperDB. In fact, HarperDB uses NATS.io for their own cluster synchronization, and can offer a more performant, featured version of this simple demo below.  

Running NATS.io in a distributed manner across Akamai Cloud Compute can demonstrate the power of distributed computing that is well connected both horizontially and vertically across public Internet, for use cases throughout IoT, gaming, media, real-time data broadcast, data collection and processing, and many more. 

## Prerequisites

* Accounts with both linode.com and Akamai
* `kubectl` and `helm` both installed (experience with Kubernetes will be helpful
* `linode-cli` installed and configured with an access token

## Building the LKE Clusters

1. The first step is to create LKE clusters in the desired regions. This example uses the linode-cli tool to do so. 

`linode-cli lke cluster-create --k8s_version 1.25 --label {label} --node_pools.count 3 --node_pools.type g6-dedicated-2 --region {regionID}`

Make note of the cluster ID that the CLI command returns. 

2. Using `linode-cli`, extract the kubeconfigs from each cluster created.

`linode-cli lke kubeconfig-view {clusterID} --json | jq -r '.[].kubeconfig | @base64d'> {regionID}`

## Deploying the Load Balancers

Before we deploy NATS.io and Prometheus (Prometheus collects and federates statistics from each NATS.io deployment), we will deploy the a load balancer service type in each cluster. This will allow us to record a two external IP addresses for each cluster (one for the NATS.io load balancer, one for the Prometheus scraping endpoint), that we will need later. 

1. Run the following command for each cluster.

`kubectl --kubeconfig={regionID} apply -f lb.yaml`

2. After the lb.yaml service is applied, query the cluster via kubectl to find the external IP address for both the Prometheus and NATS.io service for each cluster. Record these values for later.

```kubectl get svc prometheus-lb -o jsonpath="{.status.loadBalancer.ingress[*].ip}"```
```kubectl get svc natsio-lb -o jsonpath="{.status.loadBalancer.ingress[*].ip}"```








