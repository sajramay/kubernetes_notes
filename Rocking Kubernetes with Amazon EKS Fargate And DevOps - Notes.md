# Rocking Kubernetes with Amazon EKS, Fargate, And DevOps : Notes

Hand Notes from Udemy Course : https://lseg.udemy.com/course/rocking-kubernetes-with-amazon-eks-fargate-and-devops

## Kubernetes Basics

- Control Plane (k8s master) contains
  - etcd
  - scheduler
  - controller manager
  - API server

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
  - pod IPs are generally not accessible from outside the k8s cluster.  Instead Services are used to expose groups of Pods to clients

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
  
