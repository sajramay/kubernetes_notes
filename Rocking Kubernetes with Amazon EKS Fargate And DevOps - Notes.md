# Rocking Kubernetes with Amazon EKS, Fargate, And DevOps : Notes

Hand Notes from Udemy Course: https://lseg.udemy.com/course/rocking-kubernetes-with-amazon-eks-fargate-and-devops

## Kubernetes Basics

CNCF states that 51% of all Kubernetes deployments are run on AWS EKS

- Control Plane (k8s master) contains
  - etcd
    - This is a consistent and highly available key-value store that serves as Kubernetes' backing store for all cluster data.
  - kube-scheduler (Scheduler)
    - This component watches for newly created Pods that have no assigned Node and selects a Node for them to run on.
  - kube controller manager (Controller Manager)
    - This component runs various controller processes that continuously monitor the shared state of the cluster (via the API server) and make changes to move the current state towards the desired state
  - kube-apiserver (API Server)
    - This is the frontend of the Kubernetes control plane. It exposes the Kubernetes API, which is how all other control plane components, worker nodes (kubelet), and external users (via kubectl) communicate with the cluster.
  - cloud-controller-manager (Optional, for cloud environments)
    - This component integrates Kubernetes with the underlying cloud provider's API. It allows you to link your cluster into your cloud provider's infrastructure

- Worker Nodes
  - has container runtimes engines such as
    - Docker
    - Containerd
    - CRI-o
    - frakti
  - has a kubelet that runs on each node and communicates with the Control Plan
    - reports status
    - master schedules changes using the kubelet
  - has a kubeproxy
    - runs on each node in the cluster
    - enables node to node communication
    - maintains network rules
  - in AWS, Nodes are generally EC2 instances
  - each node has its own IP address
    - this is the IP address that the operating system on the node uses to communicate with other nodes in the cluster and with external networks.
    - this IP address is used by Kubernetes components like the kubelet (which runs on each node) to communicate with the Kubernetes control plane.
    - pods on different nodes can communicate with each other using their Pod IP addresses.
    - kubernetes networking aims for a "flat" network model where all Pods can communicate directly, regardless of which Node they are running on, without requiring NATs

- Pods
  - run inside nodes
  - have a defined IP address for each pod which is different from the Node IP address
  - containers within the same Pod share the same network namespace and IP address, and can communicate with each other via localhost
  - pod IPs are generally not accessible from outside the k8s cluster.  Instead, Services are used to expose groups of Pods to clients

- Replica Sets
  - ReplicaSets ensure that a specified number of identical Pods are running at all times. 
  - they handle self-healing and scaling 
  - ReplicaSets don't provide a way to manage updates to your application

- Deployments
  - set on top of ReplicaSets
  - manage the lifecycle of your application enabling
    - declarative updates to set the desired state of the application including version upgrades
    - rolling updates for a controlled rollout of a new version
      - can set maxUnavailable
    - rollbacks in case new versions have issues - k8s keeps a history of deployment revisions
    - scaling
    - self healing, inherited from the ReplicaSet
  - steps for creating a Deployment
    - create YAML with Pod template and desired replica count
    - apply using `kubectl apply -f deployment.yaml`
    - k8s API server, running on Control Plane, receives this request
    - Deployment Controller, running on Control Plane, sees the new Deployment and creates a ReplicaSet
    - changes to the Deployment lead to the creation of a *new* ReplicaSet with the old one scaled down
    - sample YAML is as follows
  ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: springldap
      labels:
        environment: test
    spec:
      replicas: 3
      selector:
        matchLabels:
          environment: test
      minReadySeconds: 10
      strategy:
        type: RollingUpdate
        rollingUpdate:
          maxSurge: 1
          maxUnavailable: 0
      template:
        metadata:
          labels:
            environment: test
        spec:
          containers:
          - name: springldap
            image: localhost:5000/dev/ramays/springldap
            ports:
              - containerPort: 9090
  ```
    - the Deployment is linked to the ReplicaSet and Pods by the environment label
    - if you manually create a Pod with the same label, this ReplicaSet will start to manage it
    - ReplicaSets restore Pods
    - Deployments restore ReplicaSets
      - if you try and delete a ReplicaSet or if one goes bad, the Deployment will recreate it
    - when you update the Deployment and `apply -f` a new ReplicaSet is created with new Pods and the old ones scaled down
      - `maxSurge` means that an additional Pod over the `replicas` count will be created

- Services
  - Pods can speak to each other using IP, but that is brittle because Pods can be replaced and IP will change
    - also, nodes can be replaced meaning that the IP will be different as well
  - Services allow you to share access to Pods which host the same container under a ReplicaSet
  - Service will automatically discover new Pods on new and existing Nodes
  - Services know which Pods to connect to based on labels
  - All Service types will discover Nodes and distribute traffic based on labels
  - Service types include
    - LoadBalancer (which maps to an ALB when running on AWS)
      - accessible from outside the cluster
      - maps to cloud provider 
      - when deployed through EKS, the associated Security Group is created and attached automatically
    - ClusterIP
      - Default type of Service, if you do not mention `kind`
      - only accessible from inside the cluster
    - Nodeport
      - attached directly to Node IP
      - you can pick port range 30000-32767
      - traffic is forwarded from `nodePort` to `port` on the Service and then onto `targetPort` on the Pod
      - for the client, the URL includes the IP of the Node but actually traffic is distributed to all matching Nodes based on `selector`
      - accessible from outside the cluster
      - uses a ClusterIP under the hood
      - for AWS, the Security Group will need to be manually created and attached
    
  - LoadBalancer YAML
  ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: springldap
      labels:
        environment: test
    spec:
      type: LoadBalancer
      ports:
      - port: 80
      selector:
        environment: test
  ```
  - Cluster IP YAML
  ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: springldap
      labels:
        environment: test
    spec:
      ports:
      - port: 80
        protocol: TCP
      selector:
        environment: test
  ```
  - Nodeport YAML
  ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: springldap
      labels:
        environment: test
    spec:
      type: NodePort
      ports:
      - nodePort: 32000
        port: 80
        targetPort: 80
        protocol: TCP
      selector:
        environment: test
  ```
  
# EKS Control Plane
  - aws managed
  - aws scales the control plane
  - aws replaces unhealthy control plane instances
  - aws maintains etcd
  - automated upgrades
  - integrated with aws ecosystem
  - flat charge of $0.10 per hour which is $72 per month

# EKS Data Plane
  - Self Managed Node Groups
    - you maintain worker ec2 machines
    - can use custom AMI
  - Managed Node Groups
    - aws manages worker ec2 instances
    - no custom AMIs
    - Note limits on number of pods depending on ec2 type and thus ENI support 
      - https://github.com/aws/amazon-vpc-cni-k8s/blob/master/misc/eni-max-pods.txt
      - https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI
  - AWS Fargate
    - no worker ec2 nodes at all
    - serverless container runtime nodes
  
#  eksctl
  - CLI Tool for creating EKS clusters
    - create, list and delete clusters
    - create, drain and delete nodegroups
    - update a cluster
    - use custom AMIs
    - configure VPC networking
    - manage iam policies
    - only works on AWS EKS
    - plus others
  - Abstracts many things like VPC,Subnets,SGs etc using CloudFormation
  - `eksctl create cluster`
    - creates a cluster with one nodegroup containing 2 m5.large nodes
  - `eksctl create cluster --name <name> --version 1.15 --node-type t3.micro --nodes 2`
    - creates a cluster with k8s version 1.15 and with 2 t3.micro nodes
  - `eksctl create cluster --name <name> --version 1.15 --nodegroup-name <nodegrpname> --node-type t3.micro --nodes 2 --managed`
    - creates a cluster with k8s version 1.15 and an aws managed node group
  - `eksctl create cluster --name <name> --fargate`
    - creates a cluster with aws fargate
  - `eksctl create nodegroup --config-file=eksctl-create-ng.yaml`
    - can create both nodegroups below from a YAML config file
    ```yaml
    ---
    apiVersoin: eksctl.io/v1alpha5
    kind: ClusterConfig
    
    metadata:
      name: eksctl-test
      region: eu-west-2
    
    nodeGroups:
      - name: ng1-public
        instanceType: t3.micro
        desiredCapacity: 2
    
    managedNodeGroups:
      - name: ng2-managed
        instanceType: t3.micro
        minSize: 1
        maxSize: 3
        desiredCapacity: 2
    ```
  - `eksctl delete cluster --name <cluster name>`

# kubectl
  - communicates with cluster using the API Server on the Control Plane
  - works on any cluster, not just AWS EKS k8s clusters
  - kubectl command syntax
    - `kubectl <command> <type> <name> <flags>`
    - command
      - create
      - get
      - describe
      - delete
      - apply
      - attach
      - autoscale
    - type (any k8s resource type)
      - pod
      - namespace
      - deployment
      - replicaset
      - service
      - plus others
    - name (optional name of the resource)
    - flags
      - `--filename` or `-f`
      - `--output` or `-o`
  - `kubectl get pod`
  - `kubectl get pod <podname>`
  - `kubectl get pod <podname> -o yaml`
  - `kubectl get ns`
    - get namespaces - this will include default namespace and system namespace
  - `kubectl get pods -n kube-system`
    - get default pods in namespace kube-system


