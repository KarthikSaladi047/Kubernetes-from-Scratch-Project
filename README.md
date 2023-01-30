# Kubernetes-from-Scratch-Project
Configuration of Kubernetes Cluster from Scratch

Note: Currently working on this project

## What is Kubernetes?

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

## Kubernetes Cluster Architecture

![components-of-kubernetes](https://user-images.githubusercontent.com/105864615/215435972-e097d12e-c7db-49f9-85a7-b5b41a625b3f.png)

👑 The Master Node components make global decisions about the cluster, as well as detecting and responding to cluster events. These components can be run on any machine in the cluster. However, for simplicity, set up scripts typically start all control plane components on the same machine, and do not run user containers on this machine.

👷 A Kubernetes cluster consists of a set of worker machines, called nodes, that run containerized applications. Every cluster has at least one worker node. The worker node(s) host the Pods that are the components of the application workload. The control plane manages the worker nodes and the Pods in the cluster.

## Set up the nodes: 
- Choose and set up the servers we will use as nodes in our cluster.

  In this project I am using a Ubuntu 22.04 vm for Master and Worker Node each. Let the hostname of Master node is "kube-master" and Worker node is "kube-worker".

  **VM Hardware Requirements:**
    <table>
      <thead>
        <tr>
          <th>Role</th>
          <th align="right">Minimal required memory</th>
          <th align="right">Minimal required CPU (cores)</th>
          <th align="right">Components</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>Master node</td>
          <td align="right">4 GB</td>
          <td align="right">2</td>
          <td align="right">kube-api-server, kube-scheduler, kube controller, etcd</td>
        </tr>
        <tr>
          <td>Worker node</td>
          <td align="right">1 GB</td>
          <td align="right">1</td>
          <td align="right">kube-proxy, kubelet, Container runtime</td>
        </tr>
      </tbody>
    </table>

## Install dependencies: 
- Install the required dependencies on each node, including Docker and other packages needed by kubernetes components.
  
## Download the binaries: 
- Download the required software components binaries, including the etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kube-proxy, kubelet and kubectl. 
 
- In this project I am using kubernetes version **v1.24.10** and etcd version **v3.5.4**.

  kubernetes download page: https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.24.md#downloads-for-v12410

  etcd download page: https://github.com/etcd-io/etcd/releases/tag/v3.5.4

**On Master Node** - Download Server Binaries and etcd Binaries
  ```
  mkdir /root/binaries
  cd /root/binaries
  wget https://dl.k8s.io/v1.24.10/kubernetes-server-linux-amd64.tar.gz
  tar -xzvf kubernetes-server-linux-amd64.tar.gz
  cd /root/binaries/kubernetes/server/bin/
  wget https://github.com/etcd-io/etcd/releases/download/v3.5.4/etcd-v3.5.4-linux-amd64.tar.gz
  tar -xzvf etcd-v3.5.4-linux-amd64.tar.gz
  ```
**On Worker Node** - Download Node Binaries
  ```
  mkdir /root/binaries
  cd /root/binaries
  wget https://dl.k8s.io/v1.24.10/kubernetes-node-linux-amd64.tar.gz
  tar -xzvf kubernetes-node-linux-amd64.tar.gz
  ```

## Pattern for Configuration of Kubernetes components

Before getting started with configuration of individual component of the kubernetes cluster, We need to decide by which patter we are configuring the cluster compontents. If we don't follow a patterned approach, it will be very difficult to establish relationship as well as secure communication between different comonents of the cluster.

There are some common set of steps that need to be performed at individual component level. They are

- **Certificate:** We will be generating a set of certificates and keys for each component for secure communication between components.
 
- **Kubeconfig:** As multiple components interact with kube-api-server, we need to generate a kubeconfig file for individual components.
 
- **Configuration of Component:** Each component of cluster will have its unique set of configuration options that control its functionality.

- **Systemd File:** We will create a systemd unit configuration file that encodes information about a process controlled and supervised by systemd. So the systemd will takes care of starting the component automatically when system reboots.

## Configure Certificate Authority

- Inorder to issue keys & certificates for individual components we need to have a Certificate Authority, which uses its own private key & Certificate to issue keys & certificates for other componets.

- We use these certificates to establish secure communication between two components. To achieve this both the components need to trust the Certificate Authority.

  ```
  mkdir /root/certificates
  cd /root/certificates
  openssl genrsa -out ca.key 2048
  openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
  openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial  -out ca.crt -days 365
  rm -f ca.csr
  ```
- Now using ca.crt and ca.key , we will issue keys & certificates for other components.

## Configure etcd: 
- Set up etcd on master node as the key-value store for the cluster.

**What is etcd ?** 
          etcd is a strongly consistent, distributed key-value store that provides a reliable way to store data that needs to be accessed by a distributed system or cluster of machines. It gracefully handles leader elections during network partitions and can tolerate machine failure, even in the leader node.

**Certificate creation**
  ```
  SERVER_IP=<your server private ip address>
  cd /root/certificates/
  openssl genrsa -out etcd.key 2048 
  cat > etcd.cnf <<EOF
  [req]
  req_extensions = v3_req
  distinguished_name = req_distinguished_name
  [req_distinguished_name]
  [ v3_req ]
  basicConstraints = CA:FALSE
  keyUsage = nonRepudiation, digitalSignature, keyEncipherment
  subjectAltName = @alt_names
  [alt_names]
  IP.1 = ${SERVER_IP}
  IP.2 = 127.0.0.1
  EOF
  openssl req -new -key etcd.key -subj "/CN=etcd" -out etcd.csr -config etcd.cnf
  openssl x509 -req -in etcd.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out etcd.crt -extensions v3_req -extfile etcd.cnf -days 365
  rm -f etcd.csr etcd.cnf
  ```
**Copy the ETCD and ETCDCTL Binaries to the Path**
  ```
  cd /root/binaries/kubernetes/server/bin/etcd-v3.5.4-linux-amd64/
  cp etcd etcdctl /usr/local/bin/
  ```
**Configure the systemd File**
  ```
  cat <<EOF | sudo tee /etc/systemd/system/etcd.service
  [Unit]
  Description=etcd
  Documentation=https://github.com/coreos

  [Service]
  ExecStart=/usr/local/bin/etcd --name master-1 --cert-file=/root/certificates/etcd.crt --key-file=/root/certificates/etcd.key --peer-cert-file=/root/certificates/etcd.crt --peer-key-file=/root/certificates/etcd.key --trusted-ca-file=/root/certificates/ca.crt --peer-trusted-ca-file=/root/certificates/ca.crt --peer-client-cert-auth --client-cert-auth --initial-advertise-peer-urls https://${SERVER_IP}:2380 --listen-peer-urls https://${SERVER_IP}:2380 --listen-client-urls https://${SERVER_IP}:2379,https://127.0.0.1:2379 --advertise-client-urls https://${SERVER_IP}:2379 --initial-cluster-token etcd-cluster-0 --initial-cluster master-1=https://${SERVER_IP}:2380 --initial-cluster-state new --data-dir=/var/lib/etcd
  Restart=on-failure
  RestartSec=5

  [Install]
  WantedBy=multi-user.target
  EOF
  ```
**Start Service**
  ```
  systemctl start etcd
  systemctl status etcd
  systemctl enable etcd
  ```
## Configure the API server: 
- Set up the kube-apiserver on one node to act as the central management component for the cluster.

## Configure the controller manager: 
- Set up the kube-controller-manager on one node to manage the state of the cluster and perform tasks such as replicating pods.

## Configure the scheduler: 
- Set up the kube-scheduler on one node to assign pods to nodes based on resource requirements and constraints.

## Configure the kubelet: 
- On each node, configure the kubelet to connect to the API server and to manage containers on the node.

## Configure the kube-proxy: 
- On each node, configure the kube-proxy to enable network connectivity for pods and to load balance network traffic to the pods.

## Join the nodes: -
- Join each node to the cluster by configuring the kubelet on each node to connect to the API server.

## Verify the cluster: 
- Verify that all nodes are healthy and that the components are running and communicating as expected.

## Deploy network add-ons: 
- Deploy network add-ons, such as a network overlay, to enable communication between pods.

## Deploy applications: 
- Deploy your desired applications to the cluster, using manifests or a continuous delivery pipeline.
