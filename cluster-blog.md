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

Pod
:  A Pod is the most basic unit of deployment in kubernetes. It encapsulates one or more containers with an IP address, a storage environment, and a set of configurations around those containers. Most of the other key abstractions in kubernetes provide ways of managing pods.

Service
:  A service defines a network access point exposed by a set of pods for other pods to consume. A service can expose one or more IP ports. The service is associated to the pods providing it by the use of a set of match rules. Typically these rules match to a label defined on the pod (often the 'app' label)

Deployment
:  A Deployment can be thought of as a declaritive template that can be used a number of pods. The Deployment allows us to manually scale the number of pods up and down (auto-scaling is handled by another abstraction).

ReplicaSet
:  A ReplicaSet controls a number of identical pods, defining the number of replicas. Often, a Deployment creates and manages a ReplicaSet under the covers

StatefulSet
:  A StatefulSet declares a number of identical pods, each of which has an identity that survives pod restarts. This allows us to assign individual storage to each of these long-lived pods and deploy stateful applications in kubernetes. My main needs here were to support kafka and cassandra (although a number of infrastructure components that I'm using for support also require state)

DaemonSet
:  A DaemonSet is used for cases where we want a single instance of a pod on each member of the kubernetes cluster. DaemonSets can be given match criteria to determine a subset of nodes to run on as well. We'll see a number of daemonsets today.

Control Plane
:  'Control plane' refers to the set of nodes that run the services that collaborate to manage the kubernetes cluster. These services include the API Server, the Controller Manager, and the Scheduler. The control plane uses etcd to store cluster metadata. The etcd cluster can be managed internally to the kubernetes cluster or externally.

## Goals
As I approached what passed for a design phase for this cluster, I kept a couple of high-level non-functional requirements in mind.
1. HA - I wanted my cluster to be 'HA-ish.' While I wasn't going to have redundant power or network and have no uptime goal, I did want the cluster to be what I've come to think of as 'maintenance tolerant,' meaning that I should be able to bring down any single node and have 'most' of the facilities remain available.
1. The cluster should employ features that are typically seen in production clusters. In particular, I wanted some redundancy in the control plane and the storage layer.
Those goals really lead in the same direction - the production features I wanted support HA.

## High level design

