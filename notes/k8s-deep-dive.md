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

# Pods
- Lifecycle: Pending, Running, Succeeded/Failed
- `restartPolicy` defines how k8s reacts to containers exiting due to errors
  - Sequence:
    - Initial crash: k8s attempts an immediate restart based on `restartPolicy`
    - Repeated crashes: Exponential backoff delay is applied for subsequent restarts
    - CrashLoopBackOff state: Container is in crash loop
    - Backoff reset: Backoff delay is resetted if container runs successfully for a certain duration
  - Possible values:
    - Always: Restarts container after any termination (default)
    - OnFailure: Only restarts container if exit code non-zero
    - Never
  - Since 1.32 the restart delay can be configured using `KubeletConfiguration` object
  - Probes:
    - liveness: Indicates whether container is running
      - if it fails kubelet kills container
    - readiness: Indicates whether container is ready to respond to requests
      - if it fails, the pod's IP is removed from all services
    - startup: Indicates whether the application within the container is started, all other probes are disabled until this one succeeds
      - if it fails kubelet kills container
  - Termination:
    - kubelet attempts graceful shutdown (SIGTERM) until grace period (default 30s) expires, then KILL signal
    - STOPSIGNAL instruction can be defined in container image to specify signal used (by default SIGTERM)
    - Since `1.33` the stop signal can also be defined as part of the container spec (dependent on OS)
    - When forceful deletes are triggered, API will remove the pod immediately without waiting for confirmation from kubelet (might run indefinetely)
    - Sidecar containers are being terminated after main containers
    - Failed pod objects remain in the API until they exceed the `terminated-pod-gc-threshold` in the kube-controller-manager
  - Init containers:
    - Start before main application container and run to completion
    - Do not support probes
    - Can run with a different view of the filesystem than app containers
  - Sidecar containers:
    - Stable since `1.33`
    - Are special init contaienrs with a restartPolicy of Always
    - Share same network and storage namespaces with the primary container and can optionally share volumes (filesystems)
  - Ephemeral Containers
    - Stable since `1.25`
    - Runs temporarily in an existing pod to aacomplish user-initiated actions such as troubleshooting
    - They lack guarantees for resources or execution and will never be automatically restarted
    - Good in addition to small images that do not provide many tools for debugging
    - Process namespaces sharing can be enabled to view processes in other containers
    - Can be added using `kubectl debug` command
  - Quality of Service (QoS) classes
    - pods are assigned to QoS classes based on the resource requests / limits
    - Guaranteed: Evicted last, memory + cpu limit = request
    - Burstable: Evicted second, not guaranteed, but at least one container has a memory/cpu request/limit
    - BestEffort: Evicted first, no memory/cpu request/limit at all
  - User namespaces
    - in beta since `1.30`
    - Security measure to isolate the user running inside the container from the one in the host
    - Processes running as root in a container can run as non-root on the host -> root only inside the user namespace
    - Granted capabilities are also only valid within the user namespace

# Deployments
  - Change causes for the revision history can be added via annotation `kubernetes.io/change-cause`
  - `progressDeadlineSeconds` can be set to tell when a deployment progress has stalled
  - `.spec.revisionHistoryLimit` can be set to specify how many old ReplicaSets to retain

# StatefulSets
  - Runs a group of pods and maintains a sticky identity for each of those pods, maintained across any rescheduling
  - Provides guarantees about ordering and uniqueness of the pods
  - Deleting a StatefulSet will not delete associated volumes
  - Require a Headless Service for the network identity
  - Pod identity consist of an ordinal, stable network identity (provided via headless service) and stable storage
    - Ordinal index is visible in pods as label `apps.kubernetes.io/pod-index`
    - Network ID:
        - Hostname = [STATEFULSET NAME]-[ORDINAL]
        - Pod DNS = [POD]-[ORDINAL].[SVC].[NS].svc.cluster.local
    - Each PVC is associated with a specific pod and will be re-attached after re-scheduling
  - Since `1.32` PVC retention can be controlled with `.spec.persistentVolumeClaimRetentionPolicy` field

# Jobs
  - `.spec.completions` can be set to specify how many pods need to complete (for parallel executions)
  - If parallel execution is set without specifying completions, a work queue can be implemented
    - Each pod nees to be capable to determine when the overall job is termated with success
    - Once a pod exits with success all other pods should be in the process of exiting
  - Pods can be restarted on failure via policy - pod stays on the node, but container is re-run -> local restarts need to be supported
  - Pods in parallel execution can be indexed for specific retry logic (since `1.33`)
  - failure and success policies can be configured - e.g. to ignore specific error codes or succeed on specific results
  - Completed jobs remain and need to be deleted manually
  - Automatic cleanup via.
    - cronjobs with capacity-based cleanup policy
    - TTL by specifying `.spec.ttlSecondsAfterFinished`
  - Cronjobs:
    - Concurrency policy can be set to allow, forbid or replace (replace old job run with new one)
    - Timezones can be specifiec by `.spec.timeZone` field (since `1.27`) - otherwise local time zone is assumed

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
- It is responsible for managing the execution and lifecycle of containers
- A working container runtime is required on each node to launch pods and containers
- More than one CR can be used within a cluster - pods can specify it via RuntimeClass
  - For each CR handler a RuntimeClass object can be created in the cluster and then referenced within the pods
  - Handlers are defined on the nodes (e.g. containerd: config.toml)
  - If a runtime is only available at specific nodes, `scheduling` field can be provided to schedule pod on corresponding nodes
  - Overhead resources when running pods in a specific runtime can be specified with `overhead` field
- Container Runtime Interface (CRI)
  - Enables kubelet to use variety of CR
  - Main protocol for communication between kubelet and container runtime (via grpc)
- Container = Technology for packaging an application along with its runtime dependencies
- Alternatives:
  - containerd
    - General purpose CR maintained by CNCF
    - Own image mgmt library, integrates well with docker images
    - Recommended by k8s as standard runtime
    - Little more overhead compared to CRI-O
    - Highly extensible with plugin architecture
  - CRI-O
    - Developed specifically for kubernetes (originally from OpenShift)
    - Focuses on minimalism and security
    - Uses containers/images and containers/storage from podman/buildah ecosystem

# Container Lifecycle Hooks
- PostStart
  - Executed immediately after container creation, but might run after ENTRYPOINT
- PreStop
  - Executed immediately before container is terminated, hook must complete before TERM signal
- Hook delivery guaranteed at least once -> Might be executed multiple times
- Hook handler implementations:
    - Exec: Executed command or script
    - HTTP: Executes HTTP request against container endpoint
    - Sleep: Pauses the container for a specified duration

# Garbage Collection
- Objects can link each other via owner references (dependent objects) - only in same namespace possible
- Cascaded deletion can be triggered to delete also dependent objects
- kubelet performs garbage collection on unused images every 5 minutes and executed based on disk usage
- kubelect garbage collects containers base on variables:
  - Min. age
  - Max. dead containers per pod
  - Max. dead containers for the cluster

# Labels, Annotations and Selectors
- Labels are intended to be used to specify identifying attributes of objects that are meaningful and relevant to users, but do not directly imply semantics to the core system
  - Labels allow for efficient queries and watches and are ideal for use in UIs and CLIs. Non-identifying information should be recorded using annotations
  - Syntax: Optional prefix needs to be DNS subdomain followed by / and the unique name per object
  - If prefix is omitted, label key is presumed to be private to the user - automated system component must specify a prefix
- Label selector core grouping mechanism in k8s
  - Can consist of multiple requirements (, separated) - translated to logical && operator, there is no logical ||
  - Types:
    - Equality-based
        - Allows filtering by label keys and values
        - Operators: =, ==, != (first two are equal)
    - Set-based
        - Allows filtering by a set of values, can be mixed with equality-based selectors
        - Operators: in, notin, exists (only for key identifier)
        - E.g.: environment in (production, qa) -> key = environment, values in (production, qa)
        - E.g.: partition -> Label with key partition exists
        - E.g.: !partition -> Labgel with key partition does not exist
- API List and Watch filtering
    - label selectors can be used to filter returned objects via query parameter
    - E.g.: ?labelSelector=environment%3Dproduction,tier%3Dfrontend
    - For kubectl -l parameter can be used: kubectl get pods -l environment=production,tier=frontend
- For labelSelector of services only equality-based selectors are allowed (while jobs, deployments, rs support both)
- Annotations
  - Attach arbitrary non-identifying metadata to objects. Clients such as tools and libraries can retrieve this metadata.
  - Annotations are not used to identify and select objects
  - Naming similar to labels (optional prefix, otherwise considered private to user)
- Field Selectors
    - Select objects based on resource fields (e.g. status.phase), but not all fields are supported
    - E.g.: `kubectl get pods --field-selector status.phase=Running`

# Autoscaling
 - In addition to hpa following scaling mechanisms can be used:
    - VerticalPodAutoscaler:
        - Separate project
        - Recreates pods when vertical scaling applies - is being worked on for in-place scaling
    - Autoscaling based on cluster size:
        - Cluster Proportional Autoscaler, separate project (beta)
    - Event-driven autoscaling
        - KEDA, separate project, CNCF
        - E.g.: Scale base on amount of messages in queue
    - Based on schedules:
        - KEDA using Cron scaler

# Networking
- Each pod in a cluster gets its own unique cluster-wide IP
- Each pod has its own private network namespace shared by all its containers that can communicate over localhost
- The pod network (also called a cluster network) handles communication between pods
- Services provide a stable (long lived) IP or hostname and act as proxies
- Gateway API / Ingresses makes service accessible outside the cluster
- Implementation of k8s networking:
  - Pod network namespaces: By CRI
  - Pod network: CNI plugins (e.g. Calico, Cilium)
  - Network policy: CNI plugins
  - Gateway API: E.g. Istio
- Network policies
  - Control traffic flow at IP address or port level (OSI 3+4)
  - Apply to a pod on at least one end of the connection
  - By default all ingress and egress connections are allowed
- DNS
  - DNS config in pods at /etc/resolv.conf - populated by kubelet
  - Pods and services receive A records
  - Named ports that are port of services receive SRV records
  - DNS policies for pods:
    - Default: Same as for the node, only named default, ist not configured as default option
    - ClusterFirst: Default, every query for outside domains is forwarded to an upstream NS
    - ClusterFirstWithHostNet: For pods running with hostNetwork
    - None: All DNS settings need to be provided in pod spec
- IPv6
  - Dual-stack is supported -> IPv4 and IPv6 addresses are configured for pods and services
  - Can be specified with `.spec.ipFamilyPolicy` on service creation
- Topology Aware Routing
  - Prefer same-zone-traffic for higher performance or lower cost
  - Hints stored in Endpoint Slices
  - Topology Aware Routing can be enabled by setting `service.kubernetes.io/topology-mode` to Auto for a Service
- Service Internal Traffic Policy
  - Keep network traffic within a node if pods are running on the same node
  - Will not route traffic at all, if there are pods only available on other nodes
  - Can be enabled by setting `.spec.internalTrafficPolicy` to Local for a service

# Services
  - Exposes an application behind a single outward-facing endpoint
  - Supported protocols: SCTP, TCP, UDP
  - Can be defined without selectors, then endpoint slices need to be added instead
  - Endpoint slices can be linked to services by setting label `kubernetes-io/service-name`
  - kubelet adds env variables for each active service
  - Since `1.33` IPAddress and ServiceCIDR objects can be used
  - Types:
    - ClusterIP: Default, k8s assigns service a cluster-internal IP address
      - Specific Cluster IP can be requested by setting `.spec.clusterIP`
      - If `.spec.clusterIP` is set to `None` then a headless service is defined
        - kube-proxy does not handle those services, no load balancing or proxying is done
        - allows client to connect to whatever pod it prefers, directly
        - report IPs of individual pods via internal DNS records
        - When defined with selectors: EndpointSlices are created and A records for the pods are returned
    - NodePort: Exposes service on each node's ip at a static port
      - Specific port can be requested
    - LoadBalancer: Exposes service externally using an external load balancer
      - k8s start requesting a Node port and the cloud-controller-manager configures the external lb to forward traffic to that port
      - `.spec.loadBalancerClass` field can be used to select a non-default lb implementation from the cloud provider
      - Sets up network-level lbs (NLB in AWS or Load Balancer in Azure) - Layer 7 lbs are used for ingress controllers
    - ExternalName: Maps service to a DNS name -> Cluster DNS will return CNAME record with external hostname value
      - No mapping via selectors but via DNS
      - Looking up [SERVICE].[NS].svc.cluster.local will return CNAME record with specified value
  - External IPs can be specified, if k8s runs behind another proxy -> service can be accessed by that IP and the defined port
  - Virtual IPs and kube-proxy
    - Every node runs kube-proxy that implements aa virtual IP for services (other than ExternalName)
    - kube-proxy configures node to capture traffic to service's cluster IP and port and redirect it to one of service's endpoints
    - Modes:
      - iptables:
        - By default select backend pod at random
        - For large cluster, lots of iptable rules can cause problems, sync. interval can be adjusted
        - Rules can be viewed on node via `sudo iptables-legacy -t nat -S | grep KUBE`
          - Each rule has a percentage value (e.g. for 3 targets): 33%, 50%, 100% and will be evaluated one after another 
      - ipvs:
        - Better performance compared to h
        - Lots of different load balancing options supported: Round Robin, Weighted Round Robin, Least Connection, WLC, Locality based Least Connection, Shortest Expected Delay, etc.
      - nftables:
        - Successor to iptables API, selects backend at random
        - Since `1.33`
  - EndpointSlices:
    - Represent a slice of the backing network endpoints for a service
    - Can contain topology information for routing purposes (node name, zone)
    - Replacement of older Endpoints API (Endpoints are mirrored to EndpointSlices for compatibility reasons) - deprecated in `1.33`

  - Traffic policies:
    - `.spec.internalTrafficPolicy` and `.spec.externalTrafficPolicy` can be used to control how traffic is routed
    - Cluster (all ready targets) vs. Local (all ready node-local targets)
  - Traffic distribution:
    - Since `1.33` to express preferences on how traffic should be routed
    - PreferClose: Preferably at same zone as client, EndpointSlices will be updated with hints
    - PreferSameZone (alpha): Will later replace PreferClose
    - PreferSameNode (alpha): Preferably at same node as client

# Ingresses
- Manage external access to the services in a cluster, typically HTTP/HTTPS (though implementations might support other protocols as well)
- May provide load balancing, SSL termination and name-based virtual hosting
- Configuration options:
  - Simple fanout: Routes traffic from single IP to more than on service based on HTTP URI requested
  - Name-based virtual hosting: Provides multiple hostnames at the same IP and routes traffic based on the hostname to different services
- Uses load balancing algorithms provided by implementation - default for nginx: Round Robin
- Cloud: Application Gateway Ingress Controller (Azure), AWS Load Balancer Controller (ALB)

# Gateway API
- Make network services available using extensible, role-oriented, protocol-aware configuration
- Supports more protocols than ingresses do (e.g. gRPC, TCP, UDP) though most of them are still experimental
- Stable APIs:
  - GatewayClass: Defines gateway type and the managing controller
  - Gateway: Defines instance of a gateway class and thus traffic handling infrastructure, such as a cloud load balancer (ALB or Application Gateway)
  - HTTPRoute: HTTP-specific rules for mapping traffic from Gateway listener to backend network endpoints (e.g. Service)

# Finalizers
- Finalizers are namespaced keys that tell Kubernetes to wait until specific conditions are met before it fully deletes resources marked for deletion. 
- Finalizers alert controllers to clean up resources the deleted object owned.
- Deletion process:
    - k8s marks object as deleted by setting metadatadeletionTimestamp
    - Control plane or other components take actions defined by finalizers and remove the corresponding finalizer afterwards
    - When the finalizers field is empty, the object gets deleted
- Finalizers don't usually specify the code to execute. Instead, they are typically lists of keys on a specific resource similar to annotations.
- E.g.: `kubernetes.io/pv-protection` Prevents deletion of PV objects when it is in use

# Owners and Dependents
- Maintained in `metadata.ownerReferences` field - e.g. Replica Set that owns pods
- Contains `blockOwnerDeletion` field that controls if dependent objects can block owner deletions
- k8s adds finalizers to an owner resource when using foreground or orphan cascading deletion

# Namespaces
- There are namespaced objects (Deployments, Services, etc.) and cluster-wide objects (StorageClass, Nodes, PersistentVolumes)
- Initial namespaces:
    - kube-node-lease: Holds lease object - e.g. for node heartbeats
    - kube-public: Readable by all clients (including those not authenticated)

# Admission Plugins
- NodeRestriction:
  - Prevents kubelets from modifying labels with node-restricition prefix for workload isolation

# Leases
- For locking shared resources, part of coordination.k8s.io API group
- Every node has its own lease in kube-node-lease namespace, every heartbeat is an update request to that lease
- Since 1.26 kube-apiserver uses leases to publish its identity

# Volumes
- Provide ability for pods tho share data via filesystem
- Use cases:
  - Populating configuration file based on a configmap or secret
  - temp space
  - sharing filesystem between two containers in the same pod
  - sharing filesystem between two different pods (even if they run on different nodes)
  - Durable storage even after restarts
- Volumes cannot be mountes nested, but similar with subPath
- Types:
  - ConfigMap:
    - Mounted as readonly and UTF-8 encoded
    - When using subPath, mount will not receive updates when ConfigMap changes
  - downwardAPI: Exposes information about the running pod
  - emptyDir:
    - Shared by all containers, will be removed when pod is removed from node
    - in memory file system can be used optionally
    - size limit can be specified
  - fc (fibre channel)
  - hostPath: Mounts file or directory from node's filesystem (security risk)
  - image (beta since `1.33`): OCI object at kubelet's host machine
  - iscsi: Existing iscsi volume can be mounted, content will be preserved, can be mounted read-only by many pods
  - local: Mounted local disk/partition/etc.
    - Can only be used as staticall-created PVs
  - nfs: Existing NFS share can be mounted
    - Possibility for multiple writers in parallel
  - persistentVolumeClaim:
    - Claim durable storage without knowing the details of the particular cloud environment
  - projected: Maps several existing volume sources into the same directory
  - secret: Always mounted as readOnly
- CSI (container storage interface) can be used to extend supported types of volumes
  - Azure: azurefile (SMB, NFS), azuredisk (block storage)
  - AWS: EBS, EFS
- Persistent Volumes
  - Provisioning Types:
    - Static
    - Dynamic: If there is no static matching PV, the cluster will try to provision one dynamically based on StorageClasses
  - PVCs are only deleted, if they are not acitvely used anymore, PVs only if they are not bound to any PVCs anymore
  - Reclaim Policy:
    - Retain: After PVC deletion the PV is not available for other pods, since there is still old data on it
    - Delete: Storage asset will be removed from external infrastructure (finalizers can be used since `1.33`)
  - PVCs can be expanded using `allowVolumeExpansion` field
  - Volume modes: Filesystem (mounted into directory) vs. block (raw block device)
  - In cloud environments PVs can be created from snapshots
  - Volume populators (controllers) can be used to create non-empty volumes, where the content is determined by a custom resource.
- Projected Volumes
  - Since `1.33` (beta) the cluster trust bundles can be injected into the container filesystem
- Ephemeral Volumes
  - Can be used if data does not need to be persisted

# Open Questions
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
- Restart policy
- Resource Quotas: https://kubernetes.io/docs/concepts/policy/resource-quotas/
- EndpointSlices
- https://kubernetes.io/docs/tasks/debug/debug-application/
- VerticalPodAutoscaler
- https://kubernetes.io/docs/reference/networking/virtual-ips/
- Services with internal load balancers

# Service Accounts
- Kubernetes will automtically inject the public root certificate and a bearer token into the pod when its initated to be able to call the api-server

# Practice
- Enable rbac for cluster
- Write own controller
- Write network policies, control which namespace can communicate with each other
- Checkout cluster trust bundles

# Progress
Finished overview, cluster architecture, containers, workloads, networking
Stopped at https://kubernetes.io/docs/concepts/storage/storage-classes/
