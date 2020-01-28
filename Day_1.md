# Day 1: Kubernetes fundamentals

## Introduction: containers in general
- Docker: simple API functionality on top of Linux namespaces and cgroups
- Containers do not inherently offer security enhancements
    - they can provide protection from common mistakes, but not against vulnerabilities
    - regular vulnerabilities found in virtualisation technologies, any process allowed to execute arbritrary code on a host could be a vulnerability for any other process
- Container namespaces are PIDs, UTS, IFS (NAT), and ____
- As containers are simply Linux processes, they share kernel-wide scheduler. Not all kernel features are namespaced, so things like memory will still refer to host-wide values
- Hypervisor overhead depends on the application load type:
    - CPU intensive tasks - ~0 overhead
    - memory intensive tasks - low overhead (~5-10%)
    - IO intensive tasks - high overhead (~100%)
- Containers benefit massively from memory utilisation compared to VMs
    - memory is usually the most expensive resource for systems
- Google developed Borg in-house and then open-sourced Kubernetes as an alternative that did not need to run in a Google data centre
    - Borg has ~80 Google full time employees working on it
    - Kubernetes has ~800 Google FTEs, nominally open-source but largely depends on community of directed actors
- Using Kubernetes does not inherently make the application in question portable
    - there will almost always be platform specific requirements e.g. storage, supporting services, that will introduce provider-specific dependencies
    - still likely to be easier than if not using Kubernetes at all though
- _Interesting aside_: there is no system call in Linux to enumerate running processes - instead the contents of `/proc` highlights what processes are runnning.
    ` ps -A` only shows processes of the container because of the _mount namespace_


## Pods
- Simplest unit of deployment in Kubernetes
- Comprised of one or more containers
- Containers within a Pod share a networking namespace, including localhost
    - have one IP between all containers run inside the Pod
    - networking namespace conflicts possible
- Rare to run Pods explicitly, instead managed by other Kubernetes objects (ReplicaSets, Deployments)


## Scheduling
- Scheduler and application both need to know the memory requirements
    - containers (as processes) can see memory limits of the whole host (as they share the kernel scheduler), so need to manage themselves
    - e.g. explicitly run a jvm with a --memory flag to limit maximum usage by single process
- As a programmer, it's critical to understand the resource footprint of your application
    - every programmer should familiarise themselves with profiling and benchmarking tools
- Pods specs:
    - Limits: safety net for other applications (neighours), info sent to kernel scheduler
    - Requests: safety net for the application process (self), info sent to Kubernetes scheduler
- Scheduling only happens _once_, at Pod creation time
- Scheduling decisions are only based on _requests_
- Killing a process is the only (unix) way of reclaiming memory
    - they cannot be throttled or partially reclaimed
- Applications should always be able to handle a restart
    - critical for running successfully in a Kubernetes environment, but should also be the case no matter where it's run
    - separation of stateless and stateful components


## Networking
- Pod IP addresses are ephemeral
- Always fronted with Services, which give stable addresses (VIPs) and cluster-namespaced DNS entries
- Services are linked to backend Pods by selectors
- Routing occurs at OSI Level 4 so highly efficient and not limited to HTTP
    - Anything that can make a TCP connection
- Services are distributed load-balancers
    - routing logic is not configurable
    - always uses a round-robin policy
    - more complex balancing requires another infrastructure layer e.g. nginx servers behind the service that route based on different parameters


## Namespaces
- A logical organnisation tool, not a security boundary
- Kubernetes objects can reference those in other namespaces
- Easy clean-up of related resources if properly namespaced, but take caution when doing so
    - delete using the `ns -[namespace]` command, not subsequent flags


## ReplicaSet
- Manages Pods
- Cannot update parameters within Pods
    - immutable once deployed


## Deployments
- Manage updates to running Pods via ReplicaSets
- Manages roll-out, deprovisioning and updates
- Supply a Pod template with image definition
- Kubernetes will handle the roll-out strategy, it's up to application and the developer to handle potentially multiple versions of the application running at once
- Always intended to be used for stateless applications (alternative is to use StatefulSet)
    - _There is no safe way to do stateful things using Deployments or ReplicaSets_
    - State must be stored elsewhere other than the cluster
- Uses a hash value of the Pod template to name Pods and deployment objects
    - changes to underlying values will always be reflected in name updates and facilitate updates
    - manifest files should also refer to image definitions that are immutable (or directly reflect file content and therefore any changes)


## Readiness and Liveness probes
- Both run by kubelet, an agent running on each node
    - checks are therefore done within cluster network space
- Liveness
    - check for responsivity
    - done per container
    - only restarts container, not the pod
- Readiness
    - check whether container is ready to receive traffic
    - container may be running but unable to connect to backend datastore or finished loading config, therefore technically responsive but not ready to serve client
    - no traffic routed to a container while readiness probe is failing
    - failing readiness probe will never restart the container
- Both checks need sensible timeouts and retries, which will vary between container role and infrastructure tier


## Storage
- Plugins for volumes
    - Defined at the Pod level (not container)
    - Containers then mount volumes at e.g. a file system path specific to that container
    - Handling concurrent access to same data store volume is an application concern (not Kubernetes)

