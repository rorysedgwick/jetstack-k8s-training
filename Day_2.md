# Day 2: Kubernetes deep-dive

## ConfigMaps and secrets
- Use to embed config and credentials within Pods
    - no version history associated with these objects within Kubernetes
- Arbitrary key:value pairs
- Can be accessed as environment variables, CLI flags or mounted as a volume across containers (from a file)
- Can be updated using e.g. `kubectl apply -f configmap.yaml` __but__ containers may not register update, depending on method of consumption
    - CLI args are evaluated at Pod start-up time
    - environment variables are not re-evaluated during process runtime
    - files can be read on initialization and/or after a modify process (using something like `inotify`), _but_ this will cause all Pods to restart simultaneously and cause service disruption
- Thefore, correct approach is define values as files, version the ConfigMap or change name, and cause a rolling update of Pods by updating the template in a Deployment
    - treating ConfigMap objects as immutable allows rolling back in case of a misconfiguration
- Secrets are stored within the Kubernetes cluster itself, with options for 3rd party integrations (for storage and fetching e.g. `git-crypt`, `Hashicorp Vault`)

## Daemonsets
- Pods that are run exactly once per node
- Used for infrastructure management tasks e.g. monitoring, logging, node configuration
- Belong to `kube-system` namespace
- Also used for Ingress objects


## Autoscaling: Application
- HorizontalPodAutoscaler is a Kubernetes object that allows scaling based on standard or custom metrics
    - standard: CPU
    - custom: requests-per-second, ...
    - usually used with a Deployment, in which case do not set a `replica` count in the HPA Deployment object
    - cannot scale to 0 Pods
- CPU-based scaling looks at `targetCPUUtilizationPercentage` average across all Pods, which is a % of _requested_ CPU
- Does not work effectively unless the Pod load is:
    - high request volume (1000s per second)
    - similarly complex requests (execution time within factor of 5 or 10)
    - request are short lived (1s - 100ms)
    - the load profile should "behave like a liquid"
- Long running processes with high resource utlisation will continue to execute on existing Pods
- Services will continue to route traffic using a round-robin approach, not targeting Pods with low utlisation
- Readiness probes can be used to see how many requests a Pod is serving and build custom logic around taking Pod out of distribution list when at a certain level
- Scaling based on custom metrics requires the application to export these values somehow (Prometheus)
    - Kubernetes metrics-server can aggregate these and take action on them
- Scaling up is typically much quicker than scaling down (~4x?)
- Scaling options may have to be configured on the Kubernetes Control Plane, and therefore may not be accessible depending on what cluster provider allows


## Autosclaling: Cluster
- Not native Kubernetes functionality (operating on the cluster), so depends on integration from support from cloud provider
- Based on number of unscheduled Pods
- Scaling up happens more readily than scaling down
- On Google Kubernetes Engine, config changes sometimes require a restart of the Control Plane, which should not cause service disruption but may be an issue if occurring during the autoscaling process
    - GKE will sometimes move your whole cluster between underlying infrastructure, depending on resource utilisation
    - Control Plane down during this time, and changes or updates won't be possible
    - you don't want an emergency during this time


## Control Plane
- AKA Master, but is a distributed system comprising multiple processes instead of single node
- Combination of dumb, stateless components (except `etcd` distributed store)
- On "Master":
    - `kubectl` is simple HTTP client
        - use `-v=9` to show the equivalent `curl` command for any `kubectl` command (without any auth parameters)
    - `apiserver` is responsible for cluster config, and is the only thing all the other resources trust or has `etcd` storage access
        - handles authentication, authorisation (RBAC), predefined and custom admission controls
    - `scheduler` will place any Pod without a `podSpec.nodeName` property onto a node, respecting placement rules
        - placement influenced by CPU, memory, node type, affinity for other pods etc
        - "ideal" placement scenario computationally impossible with even few nodes & Pods, instead best effort placement
        - "multi-dimensional game of tetris, without knowing what the next piece is"
        - sometimes unexpected behaviour, but ultimately desiged for dozens of nodes running thousands of containers
    - `controller-manager` consists of multiple controllers (something for ~every Kubernetes object type)
    - rules engine to compare and reconcile actual and desired state
- On each node:
    - `kube-proxy` creates `iptables` rules for incoming/outgoing traffic distribution among containers
        - service IP address are not actually used, as packets sent directly to IP of serving containers
    - `kubelet` listens to `apiserver` for Pod changes that match its node name
        - responsible for fetching and running containers, Readiness and Liveness probes
        - also listens for changes to other Pods, to notify `kube-proxy` of potential `iptables` updates


## Scheduling
- Can be influenced based on Pod requirements around
    - isolation
    - latency
    - node type
    - redundancy
- Can be achieved via
    - node affinity
    - Taints and Tolerations
    - Pod (anti-)affinity
- Weights need to be experimented with based on required stickyness of specific application use case
    - no hard and fast rules
- As well as user-specified weights for `preference...` rules, other placement factors contain inherent weighting
    - e.g. Services that do not want all their related Pods to run on the same node
- Affinity _based on_ Labels
- Toleration _based on_ Taints


## Ingress
- Only available for HTTP(S), which it is aware of (unlike Services, which are only aware traffic is TCP)
    - abstraction at the OSI Layer 7
- Implementation is defined by 3rd parties (not supplied by Kubernetes)
- Can use host networking & DNS names to expose nodes to external internet
- Can also use LoadBalancer Service to route traffic to specific Pods
- Ingress objects can also be used to direct internal traffic
    - this can be useful for enacting routing changes much more quickly than by other means
- `cert-manager` running inside the cluster can be used to acquire and distribute certificates for TLS termination on Ingress objects


## Storage
- Persistent Volume object is an abstraction over a named data storage object
- This can't be referenced in the Volumes section of a Pod spec template (because it references a specific storage volume, and a Pod may need to replicated)
- Persistent Volume Claims can be used to identify storage requirements that _can_ be referenced
- If there is no existing Persistent Volume to satisfy the PVC, Kubernetes will look at StorageClass objects that can specify how to create some storage resource, and then a PV object to reference it
    - this relies on 3rd party integration of StorageClass to actual storage volumes (cloud provider specific)


## StatefulSets
- `volumeClaimTemplate` defined on same level as template in the Pod spec
- Restarted Pods ("Pets") will retain same `datadir` (but will lose the root filesystem)
- StatefulSet controller is more careful about bringing new machines online to replace one that appears to be dead
- Roll-outs done in a more piecemeal manner to ensure maximum counts aren't exceeded, and hostnames are retained
- Clustering of database instances is a concern of the user (not Kubernetes)
    - specifying a mongodb instance for 3 Pods using `volumeClaimTemplate` will create 3 _individual_ mongodb instances
- Pod lifecycle hooks can be used to automate database scaling, actions to take during Pod de-provisioning etc, but these are largely not dependable
    - _WARNING_: Helm Charts look useful but be careful about running other peoples Kubernetes applications in production
    - "A good collection of badly written Kubernetes applications"