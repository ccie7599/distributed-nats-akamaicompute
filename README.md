# distributed-nats-akamaicompute

This is a cookbook for deploying a meshed, distributed NATS.io cluster across multiple Akamai Compute Regions via Linode Kubernetes Engine. 

![image](https://user-images.githubusercontent.com/19197357/225468591-5e601e39-3e74-4437-a175-4645a261be87.png)

NATS.io is a modern pub/sub protocol that can fulfill both legacy, MQTT-based use cases, as well as modern messaging applications that rely on NATS.io, or connectors to event-based data and application platforms such as HarperDB. In fact, HarperDB uses NATS.io for their own cluster synchronization, and can offer a more performant, featured version of this simple demo below.  

Running NATS.io in a distributed manner across Akamai Cloud Compute can demonstrate the power of distributed computing that is well connected both horizontially and vertically across public Internet, for use cases throughout IoT, gaming, media, real-time data broadcast, data collection and processing, and many more. 

## Prerequisites

* Accounts with both linode.com and Akamai
* `kubectl` and `helm` both installed (experience with Kubernetes will be helpful
* `linode-cli` installed and configured with an access token

## Building 

