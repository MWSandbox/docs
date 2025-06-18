# K8s deep dive

- Components:
    - ![Components](https://kubernetes.io/docs/concepts/overview/components/)

# cgroups
- Constrain resources that are allocated to processes
- Kubelet interfers with cgroups to enforce resource management of workloads
- Since 1.25 cgroup v2

# Control Plane
- CP manages the state of the cluster
- Controller manager?
  - Runs controllers to implement k8s API behavior
  - Logically each controller = separate process -> but bundled together in one single process
- Cloud controller manager?
  - Optional, integrates with underlying cloud provider
  - Contains:
    - Node controller: Update node object with cloud information, verify node's health
    - Route controller for networking
    - Service controller: Sets up load balancers and other compenents weh declaring a service that requires them
- API server?
  - Main implementation: kube-apiserver (scales horizontally)
  - Different authorization modes available:
    - ABAC, RBAC, Node, Webhook, AlwaysAllow, AlwaysDeny
    - Multiple modes can be combined (priority based on order)
- etcd (persistence store)?
  - Consistent and highly-available key value store for all API server data
- Scheduler?
  - Looks for pods not yet bound to a node and assigns each pod a suitable node

# Security
- API Server
  - Single point of contact from outside:
    - Nodes communicate via certificates (kubelet client certificates)
    - Pods communicate via service accounts
  - API server itself communicates with
    - kubelet processes
      - Use cases:
        - Fetching logs for pods
        - Attaching to running pods
        - port-forwarding
      - By default unsafe to run over public networks, since api server is not verifying kubelet certificate -> Can be changed by providing `--kubelet-certificate-authority`
      - kubelet authentication and/or authorization should be enabled
    - any node, pod or service through proxy functionality
      - Plain HTTP connections -> should not be run over public networks
  - Konnectivity service can be used to secure API server to node communication
    - Consists of server in control plane network and agents in nodes network

# Nodes
- kubelet?
  - Ensures that pods are running including their containers
  - Takes a set of PodSpecs and ensures their fulfillment
  - Can self-register a new node (default)
    - name of node = valid DNS subdomain name
    - Can be enabled / disabled via kuebelet flag
    - Node status update frequency can be specified
    - kubeconfig (credentials) need to be provided
    - Node authorization mode + NodeRestriction admission pluging enabled -> kubelet can only modify own node resource
- kube-proxy?
  - Maintains network rules on nodes to implement services

# Controllers
-  A control loop is a non-terminating loop that regulates the state of a system. In Kubernetes, controllers are control loops that watch the state of your cluster, then make or request changes where needed. Each controller tries to move the current cluster state closer to the desired state.
-  Built-in controllers run in the kube-controller-manager
- Controller pattern
  - Tracks at least one kubernetes resource type
  - Spec field of tracked objects represent the desired state
  - Controller acts to make current state come closer to desired state (either by itself or via api server)
- Node controller
  - Responsible for noticing and responding (by triggering eviction) when nodes go down
  - Assigns a CIDR block to the node when it's registered
  - Interacts with cloud provider to check if VMs are still available
- Job controller
  - Watches for Job objects that represent one-off tasks, then creates Pods to run those tasks to completion.
  - Tells api server to create/remove pods according to job spec, marks jobs as finished
- Deployment controller
  - Manages pods for a deployment
- Autoscaler controller
  - Commuinicates directly with external system to horizontally scale
- EndpointSlice controller
  - Populates EndpointSlice objects (to provide a link between Services and Pods).
- ServiceAccount controller
  - Create default ServiceAccounts for new namespaces.

# Container Runtime
- A working container runtime is required on each node to launch pods and containers
- Container Runtime Interface (CRI)
  - Enables kubelet to use variety of CR
  - Main protocol for communication between kubelet and container runtime (via grpc)

# Garbage Collection
- Objects can link each other via owner references (dependent objects) - only in same namespace possible
- Cascaded deletion can be triggered to delete also dependent objects
- kubelet performs garbage collection on unused images every 5 minutes and executed based on disk usage
- kubelect garbage collects containers base on variables:
  - Min. age
  - Max. dead containers per pod
  - Max. dead containers for the cluster

# Admission Plugins
  - NodeRestriction:
    - Prevents kubelets from modifying labels with node-restricition prefix for workload isolation

# Leases
  - For locking shared resources, part of coordination.k8s.io API group
  - Every node has its own lease in kube-node-lease namespace, every heartbeat is an update request to that lease
  - Since 1.26 kube-apiserver uses leases to publish its identity

# Open Questions
- What is a container runtime? What are the differences between different container runtimes?
- What is the operator pattern? Code an example
- https://kubernetes.io/docs/setup/production-environment/container-runtimes/
- DNS
- Backup
- Networking
- Roles
- TopologyManager feature gate (v1.27)
- Swap memory
- API Server authorization modes
- https://kubernetes.io/docs/reference/command-line-tools-reference/
- Node restriction admission plugin
- kubelet authentication and/or authorization should be enabled: https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/
- All built-in controllers
- Restart policy
- Owner references

# Service Accounts
- Kubernetes will automtically inject the public root certificate and a bearer token into the pod when its initated to be able to call the api-server

# Practice
- Enable rbac for cluster
- Write own controller

# Progress
Finished cluster architecture
Stopped at https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/