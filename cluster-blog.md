# Bare Metal Kubernetes on the Cheap
Last summer I found myself with an itch to gain a deeper understanding of how all the components in a typical kubernetes cluster work together. For me, the most natural way to gain that kind of understanding is to have an example of the machinery in question to build, take apart, and rebuild. A sensible person probably would have chosen a cloud vendor for this sort of adventure, but I've never been particularly sensible when faced with a desire to really learn anything. Since I'd been wanting a lab cluster for a while to support other odd hobbies, and really can't trust myself to tear a cloud cluster down and save myself from needing a second mortgage to pay off Amazon, I concluded that I'd build something at home.

My original plan was to get a refurbished desktop machine with a good pile of memory and set up a few VMs to support this adventure. However, two things scuppered this plan:
1. When I made my original purchase I didn't read very well - the Dell Optiplex I bought turned out to be a dual core machine - not really enough to support what I had in mind.
1. I wanted/needed to support stateful applications, both for my future workloads and to learn what was involved in supporting that sort of infrastructure.

This all lead me to the adventure at hand: building a kube cluster in my home lab on consumer-grade (i.e. cheap) hardware.

## Background
I'm assuming that this audience is at least somewhat familiar with Docker but not as familiar with Kubernetes, so I'm going to take a few minutes to give a quick overview of Kubernetes.

Kubernetes is a container orchestrator, meaning that it provides a set of tools to automate deployment, scaling, and management of containerized applications. It provides functionality to schedule workloads onto worker nodes in the cluster, detect when containers crash or exit and restart them if appropriate, establish networking between containers, connect them to distributed file storage, and a number of other handy tasks.

Kubernetes introduces a few new concepts in addition to the normal Docker world:

**Pod**
:  A Pod is the most basic unit of deployment in kubernetes. It encapsulates one or more containers with an IP address, a storage environment, and a set of configurations around those containers. Most of the other key abstractions in kubernetes provide ways of managing pods.

**Service**
:  A service defines a network access point exposed by a set of pods for other pods to consume. A service can expose one or more IP ports. The service is associated to the pods providing it by the use of a set of match rules. Typically these rules match to a label defined on the pod (often the 'app' label)

**Deployment**
:  A Deployment can be thought of as a declaritive template that can be used a number of pods. The Deployment allows us to manually scale the number of pods up and down (auto-scaling is handled by another abstraction).

**ReplicaSet**
:  A ReplicaSet controls a number of identical pods, defining the number of replicas. Often, a Deployment creates and manages a ReplicaSet under the covers

**StatefulSet**
:  A StatefulSet declares a number of identical pods, each of which has an identity that survives pod restarts. This allows us to assign individual storage to each of these long-lived pods and deploy stateful applications in kubernetes. My main needs here were to support kafka and cassandra (although a number of infrastructure components that I'm using for support also require state)

**DaemonSet**
:  A DaemonSet is used for cases where we want a single instance of a pod on each member of the kubernetes cluster. DaemonSets can be given match criteria to determine a subset of nodes to run on as well. We'll see a number of daemonsets today.

**Kubelet**
:  The kubelet daemon runs on all nodes in the cluster, managing the pods that are deployed on those nodes.

**Control Plane**
:  'Control plane' refers to the set of nodes that run the services that collaborate to manage the kubernetes cluster. These services include the API Server, the Controller Manager, and the Scheduler. The control plane uses etcd to store cluster metadata. The etcd cluster can be managed internally to the kubernetes cluster or externally.

## Goals
As I approached what passed for a design phase for this cluster, I kept a couple of high-level non-functional requirements in mind.
1. HA - I wanted my cluster to be 'HA-ish.' While I wasn't going to have redundant power or network and have no uptime goal, I did want the cluster to be what I've come to think of as 'maintenance tolerant,' meaning that I should be able to bring down any single node and have 'most' of the facilities remain available.
1. The cluster should employ features that are typically seen in production clusters. In particular, I wanted some redundancy in the control plane and the storage layer.

Those goals really lead in the same direction - the production features I wanted support HA.

## High level design
In order to accomplish those goals with a 'reasonable' investment in hardware, I decided to create a 3 node cluster where each node was a master, a worker, and a storage node.
![Diagram of infrastructure components](infra.png "Infrastructure")

In addition to the control plane components I mentioned above, this diagram also shows the storage and networking components. Starting from the bottom up, these are:
1. SATA hard disks - a single 4 TB spindle in each machine. Nice, cheap spinning rust.
1. The ceph distributed storage system, which provides configurably replicated storage.
1. The rook kubernetes operator for ceph, which integrates ceph into kubernetes in such a way that the ceph storage cluster can be managed within kubernetes. Between the two, rook and ceph allow automated provisioning of distributed storage for use by stateful sets in the kubernetes cluster
1. The flannel overlay network, allowing pod-to-pod traffic to be routed between nodes. Flannel provides a fairly simple vxlan overlay on the physical network.
1. The traefik reverse-proxy/load balancer, used as an 'ingress controller' by kubernetes. This allows clients outside the kubernetes cluster to access web applications running inside the cluster. Traefik also has integration with Let's Encrypt, providing simple management for SSL certificates for exposed applications.
1. HAProxy provides load balancing and failover outside the cluster
1. Keepalived allows an IP address to be shared between all three nodes. With HAProxy, this provides functionality similar to an F5 or ELB, decoupling external clients from physical machine IP addresses. This is critical in the failover between the API servers in the control plane.

Running on top of this core are a couple of additional peices of support infrastructure:
1. The 'ELK' stack for log aggregation and search - in this case 'ELK' is Elasticsearch, Kibana, and Filebeat (so, EFK?).
2. Prometheus and Grafana, for gathering of key performance and health indicators.

## Ansible
Ansible is a system for declaritive management of software installations. I've used it to automate the installation and initialization of the cluster so that it's repeatable. I've built the cluster at least 3 times now, thanks to this automation. It's far from perfect automation, as we'll see when we get into the procedures, but it does automate various fiddly parts of the build-out.

All the ansible (and kubernetes manifests) are available on [github](https://github.com/danch/cluster-ansible) 

## Hardware
The hardware I've built my cluster on is a bit of a grab-bag of what was economical at the time - a refurbished desktop, two or three generations of AMD processors and motherboards. The common theme (at least in terms of my intent) was 16GB or more of memory and the most cores I could get for under $500 per node. The emphasis on memory and cores has to do with my expected workloads - a number of memory intensive server processes (Kafka, Cassandra, Elasticsearch) and application loads that are highly parallel and compute intensive. Components are largely consumer grade, with the exception of the hard disks (because of the way Ceph works, disks need to be fast and support a surprisingly heavy duty cycle). Networking is 1Gb on a dedicated switch.

If I were to start over at this point, I'd settle on a single configuration, which would also include at least a small SSD for Ceph's write-ahead-log and indexes. An additional, larger SSD to provide a faster storage pool would also be interesting.

# Procedures
We're finally to the meat of the matter at hand - how to put this together and come out with a working, monitored kubernetes cluster? Most of the kubernetes set up has been derived from 

