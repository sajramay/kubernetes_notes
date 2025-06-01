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
  - Note that pods deployed to nodes in a private subnet will need a NAT in a public subnet to be able to access the internet to download from Docker Hub

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
      - larger instance types can have more ENIs attached to them
        - each ec2 instance always has at least one primary ENI
      - there is a per ENI limit of IP addresses depending on the instance type, meaning some ENIs can have more IPs than others
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

# Helm Charts

Charts are defined as instructions for navigators, so Helm instructions are called charts in keeping with the k8s nautical theme

  - helm is a package manager for k8s
  - charts are a packaging format and contain a collection of files that describe a set of k8s resources
  - they allow you to combine a number of deployments into a single file with additional parameters
  - a chart is organised as a set of files in a directory
  ```bash
  wordpress/
    Chart.yaml          # A YAML file containing information about the chart
    LICENSE             # OPTIONAL: A plain text file containing the license for the chart
    README.md           # OPTIONAL: A human-readable README file
    values.yaml         # The default configuration values for this chart
    values.schema.json  # OPTIONAL: A JSON Schema for imposing a structure on the values.yaml file
    charts/             # A directory containing any charts upon which this chart depends.
    crds/               # Custom Resource Definitions
    templates/          # A directory of templates that, when combined with values,
                        # will generate valid Kubernetes manifest files.
    templates/NOTES.txt # OPTIONAL: A plain text file containing short usage notes
  ```
  - a Chart.yaml is required for each chart and contains the following information
  ```yaml
  apiVersion: The chart API version (required)
  name: The name of the chart (required)
  version: A SemVer 2 version (required)
  kubeVersion: A SemVer range of compatible Kubernetes versions (optional)
  description: A single-sentence description of this project (optional)
  type: The type of the chart (optional)
  keywords:
    - A list of keywords about this project (optional)
  home: The URL of this projects home page (optional)
  sources:
    - A list of URLs to source code for this project (optional)
  dependencies: # A list of the chart requirements (optional)
    - name: The name of the chart (nginx)
      version: The version of the chart ("1.2.3")
      repository: (optional) The repository URL ("https://example.com/charts") or alias ("@repo-name")
      condition: (optional) A yaml path that resolves to a boolean, used for enabling/disabling charts (e.g. subchart1.enabled )
      tags: # (optional)
        - Tags can be used to group charts for enabling/disabling together
      import-values: # (optional)
        - ImportValues holds the mapping of source values to parent key to be imported. Each item can be a string or pair of child/parent sublist items.
      alias: (optional) Alias to be used for the chart. Useful when you have to add the same chart multiple times
  maintainers: # (optional)
    - name: The maintainers name (required for each maintainer)
      email: The maintainers email (optional for each maintainer)
      url: A URL for the maintainer (optional for each maintainer)
  icon: A URL to an SVG or PNG image to be used as an icon (optional).
  appVersion: The version of the app that this contains (optional). Needn't be SemVer. Quotes recommended.
  deprecated: Whether this chart is deprecated (optional, boolean)
  annotations:
    example: A list of annotations keyed by name (optional).
  ```
  - the dependency charts are defined in `dependencies` as follows
  ```yaml
  dependencies:
    - name: apache
      version: 1.2.3
      repository: https://example.com/charts
    - name: mysql
      version: 3.2.1
      repository: https://another.example.com/charts
  ```

# Scaling

## HPA Horizontal Pod Autoscaler

For a given Deployment targeting an m5.large (2vCPU and 8GB)

  ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: php-apache
    spec:
      replicas: 1
      selector:
        matchLabels:
          environment: php-apache
      template:
        metadata:
          labels:
            environment: php-apache
        spec:
          containers:
          - name: php-apache
            image: k8s.gcr.io/hpa-example
            ports:
            - containerPort: 80
            resources:
              requests:
                cpu: 500m    # same as 0.5 vCPU
                memory: 256Mi
              limits:
                cpu: 1000m   # same as 1.0 vCPU
                memory: 512Mi
  ```
and a HPA manifest
  ```yaml
    apiVersion: autoscaling/v1
    kind: HorizontalPodAutoscaler
    metadata:
      name: php-apache
      namespace: default
    spec:
      scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: php-apache
      minReplicas: 1
      maxReplicas: 10
      targetCPUUtilizationPercentage: 50
  ```
  - a new Pod will be created if a Pod CPU is > 50% of request CPU (ie > 250m)
  - HPA will increase or decrease the replica count in the deployment when it acts 
  - HPA works on actual CPU utilisation in Pods
  - HPA scales pods in a node
  - Metrics Server is needed for viewing metrics on the scaling
    - `kubectl get deployment metrics-server -n kube-system`
    - `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml`

## Cluster Auto Scaler

  - Cluster AutoScaler scales nodes in a cluster
  - it is not dependent on CPU usage, but instead based on requested CPUs
  - the actual mechanism depends on the Cloud Provider, but will be ASGs in AWS
  - there are two components
    - the open source cluster Autoscaler
    - the EKS implementation with ASG and IAM etc

## Vertical Pod Autoscaler

  - do not use in production because it discards pods
  - it should not be used with HPA 

## Karpenter

  - will automatically provision nodes of just the right size needed to host the requested pods
  - this is more efficient than provisioning multiple nodes of the same type that might just host a single pod
  - it is also faster than the Cluster Autoscaler
  - Karpenter can also provision nodes based on deployment requirements, such as GPUs, in the same node group
    - otherwise the admin would need to create a new node group which specified GPU capability
  - Karpenter needs to be installed with Helm charts and configured

# EKS Auto Mode

## DaemonSet

  - in k8s, a DaemonSet is a workload controller that ensures a copy of a Pod runs on all (or some specific) Nodes in a cluster 
    - it is a way to deploy background services that need to be present on every node
  - when a new node joins the cluster, the DaemonSet controller automatically deploys a copy of the specified Pod to that node. When a node is removed, the Pod is also automatically removed.
  - common use cases for DaemonSets include
    - Log collection agents: Tools like Fluentd or Logstash that collect logs from each node.
    - Monitoring agents: Software like Prometheus node exporter or Datadog agent that collect metrics from each node.
    - Cluster storage daemons: Components that need to run on every node to provide storage capabilities.
    - Networking proxies/helpers: Services like kube-proxy or CNI plugins that enable network functionality on each node.
  - a DaemonSet is not a pod but it does result in pods being deployed to nodes
    - the DeamonSet itself is a k8s object (controller)
  - the DaemonSet is defined in yaml and targets nodes
  ```yaml
  apiVersion: apps/v1
  kind: DaemonSet 
  ```

  
## EKS Auto Mode

  - without EKS AutoMode you need to create node groups with eks addons that are compatible with the version of EKS that you're running
    - these are apart from your normal node groups
  - with EKS AutoMode, aws will manage the core addons in the control plane
  - aws also manages the worker nodes, calling them managed instances
    - you no longer need a separate node group for the addons
      - `kubctl get nodes` will not show any nodes unless a service is deployed
    - the addons run as processes and not as daemon sets within a bespoke AMI (based on bottlerocket)

  - what is the difference between fargate and eks auto mode
    - in fargate, eks does not manage core addons which are in the control plane in eks auto mode
    - fargate does not run on ec2
    - fargate cannot run daemonset 
    - fargate cannot run on any ec2 instance type
    - eks autmode will automatically right size the instances
    - during upgrades, addons will be upgraded as well

# EKS Logging

  - EKS Control Place Logging
    - k8s api
    - audit
    - authenticator
    - controller manager
    - scheduler
  - EKS Worker Nodes Logging
    - system logs from kubelet, kube-proxy or dockerd
    - applicaton logs from application containers
    
  - Containerized applications write to 
    - stderr and stdout
  - System logs
    - systemd
  - Container redirects logs to
    - /var/log/containers/*.log
    
  - Logging agents are pods deployed using the DaemonSet
  - the logging agent will forward to the logging aggregator
  - a popular logging agents are fluentd and fluentbit
    - fluentd 
      - is written in ruby 
      - has about 100+ plugins
      - is not an official eks addon
      - can struggle with large log volume
      - instead fluentd can send to a single kinesis data firehose/kafka which can forward to other logging backends
    - fluentbit 
      - is written in C, thus is more lightweight
      - has about 20 plugins
      - is an official eks addon
      - fluentbit can send to a multiple logging backends directly
  - popular logging backends include splunk, datadog and aws elastic search
  - use kinesis data firehose to distribute to multiple backends
  - EFK stack is Elastic Search, fluentd/bit, and kibana