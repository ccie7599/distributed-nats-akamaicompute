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

## Deploying NATS.io 

NATS.io can be flexibly deployed to satisify different message and data patterns, depending on the use case. For example, a data broadcast fan-out solution of real-time sports scores will have a different design than an IoT command-and-control application that requires guaranteed message delivery. 

The design that this cookbook has the following architecture

* Single LKE clusters in each region, that host 3-pod replicasets of the NATS.io server. These pods communicate via NATS.io "cluster mode", where message data is kept strongly consistent across all partitions and subjects. More on cluster mode here - https://docs.nats.io/running-a-nats-service/configuration/clustering. 

* Each cluster connects to every other cluster via NATS.io "gateway mode." Gateway mode can be configured for various types of "interest sharing" (NATS.io terminology for topic or partition sharing), depending on the requirements of the use case. More on gateway mode here - https://docs.nats.io/running-a-nats-service/configuration/gateways.

This design provides for autoscaling ability within each region, to handle dynamic client load, along with inter-region distribution of messages in a reliable, performant manner. 

We will first create a local helm chart and add the public NATS.io helm chart as a dependency. 

1. Create a local helm chart labeled "mynats"

`helm create mynats`

2. Switch to the mynats directory `cd mynats` and update `Chart.yaml` with the following codeblock.

```
dependencies:
- name: nats
  version: 0.18.0
  repository: https://nats-io.github.io/k8s/helm/charts/
```
3. Run `helm dep update` to pull down the NATS.io helm chart locally. 

4. Create a region-specific file based on `values.yaml`. Add the following codeblock to the file. Note, the codeblock needs to be customized per-region-

* `nats.gateway.name` must be updated with a unique per-region value. 
* `nats.gateway.gateways` sections much be added for each region in use. The NATS.io load balancer IP address for each region, recorded earlier, would be used as the `nats.gateway.gateways.urls` value, in this format `- nats://{IPaddress}:7522`

The below codeblock is an example that would need to be customized with the above values. 

```
# notice the extra nats key here, must match the dependency name in Chart.yaml
nats:
  mqtt:
    enabled: true
  nats:
  cluster:
    enabled: true
    # disable cluster advertisements when running behind a load balancer
    noAdvertise: true
  gateway:
    enabled: true
    name: us-west
    noAdvertise: true
    gateways:
    - name: us-east
      urls:
      - nats://143.42.179.73:7522
    - name: us-southeast
      urls:
      - nats://139.144.164.135:7522
    - name: us-iad
      urls:
      - nats://139.144.195.154:7522
 ```
 5. We can now install NATS.io via our local helm chart and per-region values file into each cluster. 

```
helm install --kubeconfig {kubeconfig for region} {regionID} mynats -f {per-region values yaml}
```

Once each region has NATS.io installed, validation and testing can be done and is explained elsewhere in this document. 

## Installing Prometheus

The public helm chart for NATS.io includes a Prometheus exporter. Enabling collection of metrics via Prometheus is very straightforward and can be done via installation of the public helm chart.

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus   
```

Once running, metrics can be scraped via a centralized prometheus collector and server. The scraping endpoint would be ```http://{IPaddress):9090```, where `{IPAddress}` is the Prometheus load balancer IP recorded earlier. 

## Deploying a Grafana Dashboard 

## Akamai Global Traffic Management Configuration

## Testing and Troubleshooting
 







