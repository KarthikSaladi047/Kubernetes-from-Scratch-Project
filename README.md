# Kubernetes-from-Scratch-Project
Configuration of Kubernetes Cluster from Scratch

Note: Currently working on this project

## What is Kubernetes?

Kubernetes (often abbreviated as "k8s") is an open-source platform for automating the deployment, scaling, and management of containerized applications. It provides a unified way to manage containers and their associated resources, such as network and storage, allowing developers to focus on writing applications without having to worry about infrastructure.

Kubernetes is built on the concept of a cluster, which is a set of nodes running one or more containers. Nodes can be physical machines or virtual machines running in a cloud environment. The Kubernetes control plane, consisting of components such as the API server, the controller manager, and etcd, manages the state of the cluster and ensures that the desired state is maintained.

Kubernetes provides features such as self-healing, automatic scaling, and rollback and rollouts, making it an ideal platform for running large-scale, highly available applications. It also integrates with a wide range of tools and services, such as monitoring, logging, and CI/CD systems, making it a popular choice for managing cloud-native applications.

![Untitled design](https://user-images.githubusercontent.com/105864615/215563455-bc448818-7ff8-49ce-9c39-4cf4bbc2d7f1.jpg)
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

![kube cluster](https://user-images.githubusercontent.com/105864615/215555315-303fb360-360d-40e0-9bcd-0f81cd0c8e94.jpg)

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

## Pattern for Configuration of Kubernetes components:

Before getting started with configuration of individual component of the kubernetes cluster, We need to decide by which pattern we are configuring the cluster components. If we don't follow a patterned approach, it will be very difficult to establish relationship as well as secure communication between different components of the cluster.

There are some common set of steps that need to be performed at individual component level. They are

- **Certificate:** We will be generating a set of certificates and keys for each component for secure communication between components.
 
- **Kubeconfig:** As multiple components interact with kube-apiserver, we need to generate a kubeconfig file for individual components.
 
- **Configuration of Component:** Each component of cluster will have its unique set of configuration options that control its functionality.

- **Systemd File:** We will create a systemd unit configuration file that encodes information about a process controlled and supervised by systemd. So the systemd will takes care of starting the component automatically when system reboots.

## Configure Certificate Authority:

- Inorder to issue keys & certificates for individual components we need to have a Certificate Authority, which uses its own private key & Certificate to issue keys & certificates for other componets.

- We use these certificates to establish secure communication between two components. To achieve this both the components need to trust the Certificate Authority.
![PROMO-SecurityBlog-108_CloudHSM_Signed_Certs](https://user-images.githubusercontent.com/105864615/215741907-da77a5cf-b6b0-479d-a00a-94c8a4b98113.png)

  ```
  mkdir /root/certificates
  cd /root/certificates
  openssl genrsa -out ca.key 2048
  openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
  openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial  -out ca.crt -days 365
  rm -f ca.csr
  ```
- Now using ca.crt and ca.key, we will issue keys & certificates for other components.
![T513523740_g](https://user-images.githubusercontent.com/105864615/215741721-ab93ab27-ced2-47af-8bed-9b93a499aa67.jpg)
## Configure etcd: 

**What is etcd ?**

  etcd is a distributed key-value store that is used as the backend datastore for Kubernetes. It stores all the cluster configuration data, including information about the state of the cluster components and resources, such as pods, services, and nodes. The data stored in etcd is used by the Kubernetes control plane components, such as the API server and the controller manager, to enforce the desired state of the cluster.

  etcd provides a reliable and consistent way for all components of a Kubernetes cluster to access and store configuration data, making it an important component for ensuring the reliability and scalability of a Kubernetes cluster. It also provides features such as atomic transactions and watch notifications, making it a suitable choice for implementing distributed systems like Kubernetes.
Set up etcd on master node as the key-value store for the cluster.

**Certificate creation for etcd:**
  
  ```
  SERVER_IP=<your server private ip address>
  cd /root/certificates/
  {
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
  }
  openssl genrsa -out etcd.key 2048 
  openssl req -new -key etcd.key -subj "/CN=etcd" -out etcd.csr -config etcd.cnf
  openssl x509 -req -in etcd.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out etcd.crt -extensions v3_req -extfile etcd.cnf -days 365
  rm -f etcd.csr etcd.cnf
  ```
**Copy the ETCD and ETCDCTL Binaries to the Path:**
  
  ```
  cd /root/binaries/kubernetes/server/bin/etcd-v3.5.4-linux-amd64/
  cp etcd etcdctl /usr/local/bin/
  ```
**Configure the systemd File for etcd:**
  
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
**Start Service -> etcd:**
  
  ```
  systemctl start etcd
  systemctl status etcd
  systemctl enable etcd
  ```
## Configure the API server:

**What is kube-apiserver?**

  The kube-apiserver is the component of a Kubernetes cluster responsible for exposing the Kubernetes API. It provides the central interface for all cluster management functions, including create, update, and delete operations for various resources such as pods, services, and components. The kube-apiserver acts as a front-end for the Kubernetes control plane and communicates with other components such as etcd, the backend datastore, and the kube-controller-manager to ensure the desired state of the cluster is maintained. The kube-apiserver is also responsible for authentication and authorization of API requests, ensuring that only authorized entities can perform actions on the cluster.

**Certificate creation for kube-apiserver:**
  
  ```
  SERVER_IP=<your server private ip address>
  cd /root/certificates
  {
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
  }
  openssl genrsa -out kube-api.key 2048
  openssl req -new -key kube-api.key -subj "/CN=kube-apiserver" -out kube-api.csr -config kube-api.conf
  openssl x509 -req -in kube-api.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-api.crt -extensions v3_req -extfile api.conf -days 365
  rm -f kube-api.csr kube-api.conf
  ```
**Certificate Creation for Service Account:**
  
  Service Accounts in Kubernetes are used to grant access to the API server to perform actions on behalf of a particular entity, such as a user or a system component. The Service Account key pair is used to sign tokens that represent the identity of a Service Account, and are passed as part of API requests to the API server.

  By specifying the path to the Service Account private key file using the --service-account-key-file option while configuring the Kube-apiserver, the API server can use the key to sign tokens for Service Accounts and verify the signatures on incoming tokens. This ensures that only authorized Service Accounts can perform actions on the API server.

  ```
  openssl genrsa -out service-account.key 2048
  openssl req -new -key service-account.key -subj "/CN=service-accounts" -out service-account.csr
  openssl x509 -req -in service-account.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out service-account.crt -days 365
  rm -f service-account.csr
  ```
**Encryption key & EncryptionConfig yaml file Creation:**

  The EncryptionConfig file contains the configuration for encryption provider and it is requied for Kube-API-Sererver to store secrets inside etcd. So before creation of this config file we need to create a encryption key, which is used to encrypt the secrets before storing them in etcd.
  
  ```
  ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
  {
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
  }
  ```
**Copy Kube-apiserver Binaries to the Path:**
  
  ```
  cd /root/binaries/kubernetes/server/bin/
  cp kube-apiserver /usr/local/bin/
  ```
**Configure the systemd File for kube-apiserver:**
  
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
**Start Service -> kube-apiserver:**
  
  ```
  systemctl start kube-apiserver
  systemctl status kube-apiserver
  systemctl enable kube-apiserver
  ```
## Configure the controller manager:

**What is kube-controller-manager?**

  The kube-controller-manager is a component in a Kubernetes cluster that runs various controllers to manage the state of the cluster. Controllers are responsible for reconciling the actual state of the cluster with the desired state, as defined in the API server.

  For example, the replication controller, which is one of the controllers managed by the kube-controller-manager, is responsible for ensuring that the specified number of replicas of a particular pod are running in the cluster. If there are too many replicas, the replication controller will delete some pods. If there are too few replicas, the replication controller will create additional pods.

  The kube-controller-manager also manages other controllers such as the endpoints controller, which updates the endpoints of a service to reflect the current state of the pods that are part of the service, and the namespace controller, which manages the lifecycle of namespaces in the cluster.

  The kube-controller-manager runs as a separate process in the cluster, communicating with the API server to retrieve the desired state of the cluster and making changes to the actual state as necessary to ensure that it matches the desired state.

**Certificate creation for kube-controller-manager:**
  
  ```
  SERVER_IP=<your server private ip address>
  cd /root/certificates
  openssl genrsa -out kube-controller-manager.key 2048
  openssl req -new -key kube-controller-manager.key -subj "/CN=system:kube-controller-manager" -out kube-controller-manager.csr
  openssl x509 -req -in kube-controller-manager.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-controller-manager.crt -days 365
  rm -f kube-controller-manager.csr
  ```
**Copy kubectl Binaries to the Path:**
  
  Before creating the KubeConfig file for differeng components, we need to have kubectl in place.
  
  ```
  cp /root/binaries/kubernetes/server/bin/kubectl /usr/local/bin
  ```
**KubeConfig Creation for kube-controller-manager:**

  ```
  {
  kubectl config set-cluster kubernetes-from-scratch --certificate-authority=ca.crt --embed-certs=true --server=https://127.0.0.1:6443 --kubeconfig=kube-controller-manager.kubeconfig
  kubectl config set-credentials system:kube-controller-manager --client-certificate=kube-controller-manager.crt --client-key=kube-controller-manager.key --embed-certs=true --kubeconfig=kube-controller-manager.kubeconfig
  kubectl config set-context default --cluster=kubernetes-from-scratch --user=system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig   
  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
  }
  ```
**Copy kube-controller-manager Binaries to the Path:**

  ```
  cd /root/binaries/kubernetes/server/bin/
  cp kube-controller-manager /usr/local/bin/
  ```
**Configure the systemd File for kube-controller-manager:**
  
  ```
  cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
  [Unit]
  Description=Kubernetes Controller Manager
  Documentation=https://github.com/kubernetes/kubernetes

  [Service]
  ExecStart=/usr/local/bin/kube-controller-manager --bind-address=0.0.0.0 --service-cluster-ip-range=10.32.0.0/24 --cluster-cidr=10.200.0.0/16 --kubeconfig=/root/certificates/kube-controller-manager.kubeconfig --authentication-kubeconfig=/root/certificates/kube-controller-manager.kubeconfig --authorization-kubeconfig=/root/certificates/kube-controller-manager.kubeconfig --leader-elect=true --cluster-signing-cert-file=/root/certificates/ca.crt --cluster-signing-key-file=/root/certificates/ca.key --root-ca-file=/root/certificates/ca.crt --service-account-private-key-file=/root/certificates/service-account.key --use-service-account-credentials=true --v=2
  Restart=on-failure
  RestartSec=5

  [Install]
  WantedBy=multi-user.target
  EOF
  ```
**Start Service -> kube-controller-manager:**
  
  ```
  systemctl start kube-controller-manager
  systemctl status kube-controller-manager
  systemctl enable kube-controller-manager
  ```

## Configure the scheduler: 

**What is kube-scheduler?**

  The kube-scheduler is a component in a Kubernetes cluster that assigns work to individual nodes. The kube-scheduler decides which node a newly created pod should run on, based on the available resources on each node and the constraints specified for the pod.

  The kube-scheduler uses information about the state of the cluster, such as the CPU and memory utilization of each node, to make its scheduling decisions. It also takes into account various constraints specified by the user, such as required network access, affinity and anti-affinity rules, and node labels.

  The kube-scheduler runs as a separate process in the cluster and communicates with the API server to retrieve information about pods that need to be scheduled and the state of the nodes in the cluster. When a scheduling decision is made, the kube-scheduler updates the API server with the chosen node for the pod, and the kubelet on the selected node is responsible for pulling the pod and starting it.

  By managing the assignment of work to nodes in the cluster, the kube-scheduler helps ensure that the cluster's resources are used effectively and efficiently, and that pods are placed on nodes that have the necessary resources to run them.

**Certificate creation for kube-scheduler:**
  
  ```
  SERVER_IP=<your server private ip address>
  cd /root/certificates
  openssl genrsa -out kube-scheduler.key 2048
  openssl req -new -key kube-scheduler.key -subj "/CN=system:kube-scheduler" -out kube-scheduler.csr
  openssl x509 -req -in kube-scheduler.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-scheduler.crt -days 365
  rm -f kube-scheduler.csr
  ```
**KubeConfig Creation for kube-scheduler:**

  ```
  cd /root/certificates
  {
  kubectl config set-cluster kubernetes-from-scratch --certificate-authority=ca.crt --embed-certs=true --server=https://127.0.0.1:6443 --kubeconfig=kube-scheduler.kubeconfig
  kubectl config set-credentials system:kube-scheduler --client-certificate=kube-scheduler.crt --client-key=kube-scheduler.key --embed-certs=true --kubeconfig=kube-scheduler.kubeconfig
  kubectl config set-context default --cluster=kubernetes-from-scratch --user=system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
  }
  ```
**Copy kube-scheduler Binaries to the Path:**

  ```
  cd /root/binaries/kubernetes/server/bin/
  cp kube-scheduler /usr/local/bin/
  ```
**Configure the systemd File for kube-scheduler:**
  
  ```
  cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
  [Unit]
  Description=Kubernetes Scheduler
  Documentation=https://github.com/kubernetes/kubernetes

  [Service]
  ExecStart=/usr/local/bin/kube-scheduler --kubeconfig=/root/certificates/kube-scheduler.kubeconfig --authentication-kubeconfig=/root/certificates/kube-scheduler.kubeconfig --authorization-kubeconfig=/root/certificates/kube-scheduler.kubeconfig --bind-address=127.0.0.1 --leader-elect=true
  Restart=on-failure
  RestartSec=5

  [Install]
  WantedBy=multi-user.target
  EOF
  ```
**Start Service -> kube-scheduler:**
  
  ```
  systemctl start kube-scheduler
  systemctl status kube-scheduler
  systemctl enable kube-scheduler
  ```

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
