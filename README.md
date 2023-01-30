# Kubernetes-from-Scratch-Project
Configuration of Kubernetes Cluster from Scratch

Note: Currently working on this project

#What is Kubernetes?

Kubernetes, also known as K8s, is an open source system for managing containerized applications across multiple hosts. It provides basic mechanisms for deployment, maintenance, and scaling of applications. Kubernetes is hosted by the Cloud Native Computing Foundation (CNCF).

KeY Features of Kubernetes are:

- Automated rollouts and rollbacks
- Service discovery and load balancing
- Storage orchestration
- Self-healing
- Secret and configuration management
- Automatic bin packing
- Batch execution
- Horizontal scaling
- IPv4/IPv6 dual-stack
- Designed for extensibility


Official Kubernetes Documentation: https://kubernetes.io/
Kubernetes Project GitHub: https://github.com/kubernetes/kubernetes

#Kubernetes Cluster Architecture

![components-of-kubernetes](https://user-images.githubusercontent.com/105864615/215426841-8bdcac7d-6219-4eb2-bc58-92e08003cada.png)

The Master Node components make global decisions about the cluster, as well as detecting and responding to cluster events.

These components can be run on any machine in the cluster. However, for simplicity, set up scripts typically start all control plane components on the same machine, and do not run user containers on this machine.

A Kubernetes cluster consists of a set of worker machines, called nodes, that run containerized applications. Every cluster has at least one worker node.

The worker node(s) host the Pods that are the components of the application workload. The control plane manages the worker nodes and the Pods in the cluster.

#Set up the nodes: Choose and set up the servers we will use as nodes in our cluster.

In this project I am using a Ubuntu 22.04 vm for Master and Worker Node each. Let the hostname of Master node is "kube-master" and Worker node is "kube-worker".

##VM Requirements:
<table><thead><tr><th>Role</th><th align="right">Minimal required memory</th><th align="right">Minimal required CPU (cores)</th><th align="right">Components</th></tr></thead><tbody><tr><td>Master node</td><td align="right">2 GB</td><td align="right">1.5</td><td align="right">Kublr-Kubernetes master components (k8s-core, cert-updater, fluentd, kube-addon-manager, rescheduler, network, etcd, proxy, kubelet)</td></tr><tr><td>Worker node</td><td align="right">700 mB</td><td align="right">0.5</td><td align="right">Kubernetes worker components (dns, proxy, network, kubelet)</td></tr></tbody></table>

#Install dependencies: Install the required dependencies on each node, including Docker and other packages needed by kubernetes components.

  
#Download the binaries: Download the required software components, including the etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kube-proxy, kubelet and kubectl.

##On Master Node - Download Server Binaries
```
mkdir /root/binaries
cd /root/binaries
wget https://dl.k8s.io/v1.24.1/kubernetes-server-linux-amd64.tar.gz
tar -xzvf kubernetes-server-linux-amd64.tar.gz
cd /root/binaries/kubernetes/server/bin/
wget https://github.com/etcd-io/etcd/releases/download/v3.5.4/etcd-v3.5.4-linux-amd64.tar.gz
tar -xzvf etcd-v3.5.4-linux-amd64.tar.gz
```
##On Worker Node - Download Node Binaries
```
mkdir /root/binaries
cd /root/binaries
wget https://dl.k8s.io/v1.24.1/kubernetes-node-linux-amd64.tar.gz
tar -xzvf kubernetes-node-linux-amd64.tar.gz
```
#Configure etcd: Set up etcd on one or more nodes as the key-value store for the cluster.

#Configure the API server: Set up the kube-apiserver on one node to act as the central management component for the cluster.

#Configure the controller manager: Set up the kube-controller-manager on one node to manage the state of the cluster and perform tasks such as replicating pods.

#Configure the scheduler: Set up the kube-scheduler on one node to assign pods to nodes based on resource requirements and constraints.

#Configure the kubelet: On each node, configure the kubelet to connect to the API server and to manage containers on the node.

#Configure the kube-proxy: On each node, configure the kube-proxy to enable network connectivity for pods and to load balance network traffic to the pods.

#Join the nodes: Join each node to the cluster by configuring the kubelet on each node to connect to the API server.

#Verify the cluster: Verify that all nodes are healthy and that the components are running and communicating as expected.

#Deploy network add-ons: Deploy network add-ons, such as a network overlay, to enable communication between pods.

#Deploy applications: Deploy your desired applications to the cluster, using manifests or a continuous delivery pipeline.
