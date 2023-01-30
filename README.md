# Kubernetes-from-Scratch-Project
Configuration of Kubernetes Cluster from Scratch

Note: Currently working on this project

## What is Kubernetes?

Kubernetes, also known as K8s, is an open source system for managing containerized applications across multiple hosts. It provides basic mechanisms for deployment, maintenance, and scaling of applications. Kubernetes is hosted by the Cloud Native Computing Foundation (CNCF).

Key Features of Kubernetes are:

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

Kubernetes GitHub: https://github.com/kubernetes/kubernetes

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
          <td align="right">kube-apiserver, kube-scheduler, kube controller, etcd</td>
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

  Kubernetes Binaries Download Page: https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.24.md#downloads-for-v12410

  etcd Binaries Download Page: https://github.com/etcd-io/etcd/releases/tag/v3.5.4

  **Server Binaries and etcd Binaries** - Download Server Binaries and etcd Binaries on Master Node

    ```
    mkdir /root/binaries
    cd /root/binaries
    wget https://dl.k8s.io/v1.24.10/kubernetes-server-linux-amd64.tar.gz
    tar -xzvf kubernetes-server-linux-amd64.tar.gz
    cd /root/binaries/kubernetes/server/bin/
    wget https://github.com/etcd-io/etcd/releases/download/v3.5.4/etcd-v3.5.4-linux-amd64.tar.gz
    tar -xzvf etcd-v3.5.4-linux-amd64.tar.gz
    ```
  **Node Binaries** - Download Node Binaries on Worker Node

    ```
    mkdir /root/binaries
    cd /root/binaries
    wget https://dl.k8s.io/v1.24.10/kubernetes-node-linux-amd64.tar.gz
    tar -xzvf kubernetes-node-linux-amd64.tar.gz
    ```

## Pattern for Configuration of Kubernetes components

Before getting started with configuration of individual component of the kubernetes cluster, We need to decide by which pattern we are configuring the cluster components. If we don't follow a patterned approach, it will be very difficult to establish relationship as well as secure communication between different components of the cluster.

There are some common set of steps that need to be performed at individual component level. They are

- **Certificate:** We will be generating a set of certificates and keys for each component for secure communication between components.
 
- **Kubeconfig:** As multiple components interact with kube-apiserver, we need to generate a kubeconfig file for individual components.
 
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

**What is etcd ?**

  etcd is a strongly consistent, distributed key-value store that provides a reliable way to store data that needs to be accessed by a distributed system or cluster of machines. It gracefully handles leader elections during network partitions and can tolerate machine failure, even in the leader node.

Set up etcd on master node as the key-value store for the cluster.

**Certificate creation for etcd**
  
  ```
  SERVER_IP=<your server private ip address>
  cd /root/certificates/
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
  openssl genrsa -out etcd.key 2048 
  openssl req -new -key etcd.key -subj "/CN=etcd" -out etcd.csr -config etcd.cnf
  openssl x509 -req -in etcd.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out etcd.crt -extensions v3_req -extfile etcd.cnf -days 365
  rm -f etcd.csr etcd.cnf
  ```
**Copy the ETCD and ETCDCTL Binaries to the Path**
  
  ```
  cd /root/binaries/kubernetes/server/bin/etcd-v3.5.4-linux-amd64/
  cp etcd etcdctl /usr/local/bin/
  ```
**Configure the systemd File for etcd**
  
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
**Start Service -> etcd**
  
  ```
  systemctl start etcd
  systemctl status etcd
  systemctl enable etcd
  ```
## Configure the API server:

**What is kube-apiserver?**

  The kube-apiserver is the component of a Kubernetes cluster responsible for exposing the Kubernetes API. It provides the central interface for all cluster management functions, including create, update, and delete operations for various resources such as pods, services, and components. The kube-apiserver acts as a front-end for the Kubernetes control plane and communicates with other components such as etcd, the backend datastore, and the kube-controller-manager to ensure the desired state of the cluster is maintained. The kube-apiserver is also responsible for authentication and authorization of API requests, ensuring that only authorized entities can perform actions on the cluster.
  
Set up the kube-apiserver on one node to act as the central management component for the cluster.

**Certificate creation for kube-apiserver**
  
  ```
  SERVER_IP=<your server private ip address>
  cd /root/certificates
  cat <<EOF | sudo tee kube-api.conf
  [req]
  req_extensions = v3_req
  distinguished_name = req_distinguished_name
  [req_distinguished_name]
  [ v3_req ]
  basicConstraints = CA:FALSE
  keyUsage = nonRepudiation, digitalSignature, keyEncipherment
  subjectAltName = @alt_names
  [alt_names]
  DNS.1 = kubernetes
  DNS.2 = kubernetes.default
  DNS.3 = kubernetes.default.svc
  DNS.4 = kubernetes.default.svc.cluster.local
  IP.1 = 127.0.0.1
  IP.2 = ${SERVER_IP}
  IP.3 = 10.32.0.1
  EOF
  openssl genrsa -out kube-api.key 2048
  openssl req -new -key kube-api.key -subj "/CN=kube-apiserver" -out kube-api.csr -config kube-api.conf
  openssl x509 -req -in kube-api.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-api.crt -extensions v3_req -extfile api.conf -days 365
  rm -f kube-api.csr kube-api.conf
  ```
**Certificate Creation for Service Account**
  
  Service Accounts in Kubernetes are used to grant access to the API server to perform actions on behalf of a particular entity, such as a user or a system component. The Service Account key pair is used to sign tokens that represent the identity of a Service Account, and are passed as part of API requests to the API server.

  By specifying the path to the Service Account private key file using the --service-account-key-file option while configuring the Kube-apiserver, the API server can use the key to sign tokens for Service Accounts and verify the signatures on incoming tokens. This ensures that only authorized Service Accounts can perform actions on the API server.

  ```
  openssl genrsa -out service-account.key 2048
  openssl req -new -key service-account.key -subj "/CN=service-accounts" -out service-account.csr
  openssl x509 -req -in service-account.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out service-account.crt -days 365
  rm -f service-account.csr
  ```
**Encryption key & EncryptionConfig yaml file Creation**
  This file contains the configuration for encryption provider and it is requied for Kube-API-Sererver to store secrets inside etcd. So before creation of this config file we need to create a encryption key, which is used to encrypt the secrets before storing them in etcd.
  
  ```
  ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
  cat > encryption-at-rest.yaml <<EOF
  kind: EncryptionConfig
  apiVersion: v1
  resources:
    - resources:
        - secrets
      providers:
        - aescbc:
            keys:
              - name: key1
                secret: ${ENCRYPTION_KEY}
        - identity: {}
  EOF
  ```
**Copy Kube-apiserver Binaries to the Path**
  
  ```
  cd /root/binaries/kubernetes/server/bin/
  cp kube-apiserver /usr/local/bin/
  ```
**Configure the systemd File for kube-apiserver**
  
  ```
  cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
  [Unit]
  Description=Kubernetes API Server
  Documentation=https://github.com/kubernetes/kubernetes

  [Service]
  ExecStart=/usr/local/bin/kube-apiserver --advertise-address=${SERVER_IP} --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/root/certificates/ca.crt --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota --enable-bootstrap-token-auth=true --etcd-cafile=/root/certificates/ca.crt --etcd-certfile=/root/certificates/etcd.crt --etcd-keyfile=/root/certificates/etcd.key --etcd-servers=https://127.0.0.1:2379 --kubelet-client-certificate=/root/certificates/kube-api.crt --kubelet-client-key=/root/certificates/kube-api.key --service-account-key-file=/root/certificates/service-account.crt --service-cluster-ip-range=10.32.0.0/24 --tls-cert-file=/root/certificates/kube-api.crt --tls-private-key-file=/root/certificates/kube-api.key --requestheader-client-ca-file=/root/certificates/ca.crt --service-node-port-range=30000-32767 --audit-log-maxage=30 --audit-log-maxbackup=3 --audit-log-maxsize=100 --audit-log-path=/var/log/kube-api-audit.log --bind-address=0.0.0.0 --event-ttl=1h --service-account-key-file=/root/certificates/service-account.crt --service-account-signing-key-file=/root/certificates/service-account.key --service-account-issuer=https://${SERVER_IP}:6443 --encryption-provider-config=/root/certificates/encryption-at-rest.yaml --v=2
  Restart=on-failure
  RestartSec=5

  [Install]
  WantedBy=multi-user.target
  EOF
  ```
**Start Service -> kube-apiserver**
  
  ```
  systemctl start kube-apiserver
  systemctl status kube-apiserver
  systemctl enable kube-apiserver
  ```
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
