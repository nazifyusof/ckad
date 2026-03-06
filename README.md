<p align="center"><img width="180" alt="portfolio_view" src="badge.png"></p>
<h4 align="center"><a href="https://www.cncf.io/certification/ckad/">https://www.cncf.io/certification/ckad/</a></h3>
<h1 align="center">Certified Kubernetes Application Developer (CKAD) Notes</h1>

# CKAD Practice
- https://learn.kodekloud.com/user/courses/udemy-labs-certified-kubernetes-administrator-with-practice-tests 
- 

# CKAD Notes

## 1. Core Concepts

### Components
- Master Nodes - Manage, plan, schedule, and monitor nodes
  - Kube-apiserver - API server for Kubernetes cluster
    - etcd - Key-value store for Kubernetes cluster data
    - Kube-Scheduler - Schedules pods to worker nodes based on resource availability and constraints
    - Controller-manager - Runs controllers that manage the state of the cluster
        - Node-controller - Monitors node health and manages node lifecycle
        - Replication-controller - Ensures the desired number of pod replicas are running
- Worker Nodes - Host applications as containers
  - Kubelet (captain of the ship) - Agent that runs on each worker node, responsible for managing pods and containers
  - Kube-proxy - Network proxy that runs on each worker node, responsible for maintaining network rules and load balancing
  - Container Runtime Engine - Software that runs containers (e.g. containerd, rkt and etc.)

### etcd 
- Distributed key-value store that stores all cluster data
- Steps to start etcd:
  1. Download binaries
  2. Extract 
  3. Run etcd server on port 2379
- `etcd client` can be used to interact with the `etcd server` to perform operations like setting and getting key-value pairs
- Deploy multiple etcd servers for high availability
- setup with `kubeadm` - deploys the etcd server as pods in the cluster, managed by the kube-apiserver
  - to list all keys: `kubectl exec etcd-master -n kube-system etcdctl get / --prefix --keys-only`

### kube-apiserver
- with `kubeadm`
  - View apiserver: kubectl get pods -n kube-system`, and look for `kube-apiserver-*` pods
  - view apiserver options: `cat /etc/kubernetes/manifests/kube-apiserver.yaml`
- with non-`kubeadm`
  - view apiserver options
    - `cat /etc/systemd/system/kube-apiserver.service`, or
    - `ps -aux | grep kube-apiserver` to find the process and its options

### kube-controller-manager
- What it does:
  - Watch Status
  - Remediate situations
  - Monitor period: 5s
  - Monitor Grace period: 40s
  - Eviction timeout: 5m
- with `kubeadn`
  - view controller-manager: `kubectl get pods -n kube-system`, and look for `kube-controller-manager-*` pods
  - View options: `cat /etc/kubernetes/manifests/kube-controller-manager.yaml`
- With non-`kubeadm`
  - view controller-manager options
    - `cat /etc/systemd/system/kube-controller-manager.service`, or
    - `ps -aux | grep kube-controller-manager` to find the process and its options

### kube-scheduler
- decides which pod goes where
- with `kubeadm`
  - view scheduler: `kubectl get pods -n kube-system`, and look for `kube-scheduler-*` pods
  - View options: `cat /etc/kubernetes/manifests/kube-scheduler.yaml`
- with non-`kubeadm`
  - view scheduler options
    - `cat /etc/systemd/system/kube-scheduler.service`, or
    - `ps -aux | grep kube-scheduler` to find the process and its options

### kubelet
> [!IMPORTANT]
> kubeadm DOES NOT automatically deploy kubelets
- Always manually install kubelet on each worker node, `wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubelet`
- View kubelet options
  - `ps -aux | grep kubelet` to find the process and its options

### kube-proxy
- kube-proxy is a process that runs on each node in the cluster
- Everytime a new service is created, kube-proxy updates the iptables rules to route traffic to the appropriate pods
- Every pod can reach other pods, by deploying a pod networking solution e.g. webapp on pod 1 and database on pod 2
- with `kubeadm`
  - it deploys kube-proxy as a DaemonSet, which ensures that there is a kube-proxy pod running on each node in the cluster
  - view kube-proxy: `kubectl get pods -n kube-system`, and look for `kube-proxy-*` pods
  - view daemonset: `kubectl get daemonset -n kube-system`, and look for `kube-proxy` daemonset

### pods
- Smallest deployable unit in Kubernetes
- To scale, we usually don't increase containers in a pod, we increase the number of pods instead

#### multi-container pods (with helpers):
- Sidecar - A helper container that enhances the main container, e.g. log collector, proxy, etc.
- With pods, we don't need to maintain the mapping of which helper container belongs to which main container, because they are all in the same pod and share the same lifecycle

#### Commands
- Deploy a pod: `kubectl run <pod-name> --image=<image-name>`
- View pods: `kubectl get pods`
- Describe pods: `kubectl describe pod <pod-name>`
- Create file below to setup pod configs
```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: myapp-pod
  labels:
    app: myapp
    type: front-end        # <-- can have any kind of key-value pair as we see fit, but we should follow some conventions for better organization and management
spec: 
  containers:
    - name: nginx-container
      image: nginx:latest
```
- create the pod: `kubectl create -f pod-definition.yaml`

## 2. Configuration
### Kind
| Kind | Version |
| --- | --- |
| Pod | v1 |
| Service | v1 |
|ReplicaSet | apps/v1 |
| Deployment | apps/v1 |

### ReplicaSet
- Reason
    - high availability
    - Load balancing and scaling
  
#### Replication-controller (Deprecated)
#### ReplicaSet
- requires a selector! why?
  - can also manage pods that were not created by the ReplicaSet, as long as they match the selector
  - use the same label we use to create the pods, so that the ReplicaSet can manage those pods

 ```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
  labels:
    app: myapp
    type: front-end        # <-- can have any kind of key-value pair as we see fit, but we should follow some conventions for better organization and management
spec:
  template:
    metadata:
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
        - name: nginx-container
          image: nginx:latest
  replicas: 3
  selector: 
    matchLabels:
      type: front-end
```

#### Commands
- Deploy a ReplicaSet: `kubectl create -f replicaset-definition.yaml`
- View ReplicaSet: `kubectl get replicaset`
- To update: `kubectl apply -f replicaset-definition.yaml`
- To delete: `kubectl delete replicaset <replicaset-name>`
- To scale: `kubectl scale --replicas=<number-of-replicas> -f replicaset-definition.yaml` or `kubectl scale --replicas=<number-of-replicas> replicaset <replicaset-name>` (not updated in the file)

### Deployments

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
    type: front-end        # <-- can have any kind of key-value pair as we see fit, but we should follow some conventions for better organization and management
spec:
  template:
    metadata:
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
        - name: nginx-container
          image: nginx:latest
  replicas: 3
  selector:
    matchLabels:
      type: front-end
```
#### Commands
- create or update the deployment: `kubectl create -f deployment-definition.yaml`
- get deployment: `kubectl get deployments`
- get replicaset: `kubectl get replicaset`
- get pods: `kubectl get pods`
- get all objects at once: `kubectl get all`
- expose deployment: `kubectl expose deployment <deployment-name> --port=<service-port>`


### Services
- Enable communication between pods and other services, both inside and outside the cluster
- We don't want to ssh into the node, we want to access the application running in the cluster from outside, so we need to expose the service to the outside world
- How can we achieve? 
  - **NodePort**: Exposes the service on a static port on each node's IP address. External users can access the service using `<NodeIP>:<NodePort>`.
    - **Target port**: The port on which the application is running inside the pod
    - **Service port**: The port on which the service is exposed to the outside world
    - **Node port**: The port on which the service is exposed on each node's IP address, range: 30000-32767
  - **ClusterIP**: Exposes the service on a cluster-internal IP. This is the default type and is only accessible within the cluster.
  - **LoadBalancer**: Provisions an external load balancer that routes traffic to the service. This is typically used in cloud environments.
- For multiple pods, no extra step is needed. The service will automatically load balance the traffic between the pods that match the selector. 

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: NodePort
  ports:
  - port: 80          # service port (service)
    targetPort: 80    # target port (pod)
    nodePort: 30080   # node port (node)
  selector:
    app: myapp
    type: front-end
```
- create or update the service: `kubectl create -f service-definition.yaml`
- get services: `kubectl get services`, (see the type and ports)

#### Service ClusterIP
- Cannot rely on the current IP since they are static and can change if the service is deleted and recreated
- ClusterIP helps us to avoid this issue by providing a stable IP address that can be used to access the service within the cluster, regardless of the underlying pod IPs or node IPs. The ClusterIP is assigned to the service and remains constant as long as the service exists, allowing other pods to reliably communicate with it using the ClusterIP.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  ports:
    - port: 80          # service port (service)
      targetPort: 80    # target port (pod)
  selector:
    app: myapp
    type: front-end
```
- create or update the service: `kubectl create -f service-definition.yaml`
- get services: `kubectl get services`, (see the type and ports)

#### Service Load balancer
- User need single URL to access the service
- We can create a new VM to install a load balancer, but it's easier to use the cloud provider's load balancer, which is automatically provisioned when we create a service of type LoadBalancer e.g. GCP, AWS, Azure, etc.
- All we need to do is below:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: LoadBalancer
  ports:
    - port: 80          # service port (service)
      targetPort: 80    # target port (pod)
      nodePort: 30008   # node port (node)
```

### Namespaces
- `<service-name>.<namespace>.svc.cluster.local` is the full DNS name for a service
- create a pod in a specific namespace: `kubectl run nginx --image=nginx -n <namespace-name>` or below
```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: myapp-pod
  namespace: <namespace-name>
  labels:
    app: myapp
    type: front-end
spec: 
  containers:
    - name: nginx-container
      image: nginx:latest
```
- Create namespace: `kubectl create namespace <namespace-name>` or `kubectl create -f namespace-definition.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
    name: <namespace-name>
```
- Change current context to a specific namespace: `kubectl config set-context $(kubectl config current-context) --namespace=<namespace-name>`


### Imperative vs Declarative

#### Imperative (save time in ckad exam, but not recommended for production)
- Create onject
  - kubectl craete -f nginx.yaml
- Update Objects Imperatively
  - kubectl edit - kubectl nginx (opens the object in the default editor, allows you to make changes and save, which will update the object in the cluster)
  - kubectl replace -f nginx.yaml (deletes and recreates the object, which can cause downtime)

#### Declarative
- Create or update objects: 
  - `kubectl apply -f nginx.yaml`, create object if it doesn't exist, or update it if it does exist
  - `kubectl apply -f /filepath/` if multiple, to apply all YAML files in the directory
- Update 
  - `kubectl apply -f nginx.yaml`

## 3. Scheduling
- Scheduling is the process of assigning pods to nodes based on resource availability and constraints
- The kube-scheduler is responsible for scheduling pods to nodes

### Manual scheduling
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: <name-of-the-node> # <-- this will schedule the pod to the specified node, but it is not recommended for production as it does not provide any flexibility or fault tolerance
  containers:
  - image: nginx
    name: nginx
```

### Labels and Selectors
- We can group object by their types, application, functionality, etc. using labels and selectors

#### Command
- Filter objects by selectors, `kubectl get pods --selector env=dev`
- Filter objects by multiple selectors, `kubectl get pods --selector env=dev,tier=frontend`

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: simple-webapp
  labels:
    name: App1                 # this is the labels configured on the replicaset, used to configure object to discover the replicaset
    function: front-end
  annotations:
    buildversion: "1.0"          # record other details for better management, not used for scheduling or selection
spec:
  replicas: 2
  selector:
    matchLabels:
      app: App1                    # this is the selector used to select the pods that belong to this replicaset, it should match the labels configured on the pod
  template:
    metadata:
      labels:
        name: App1                  # this is the labels configured on the pod
        function: front-end
    spec:
      containers:
      - name: simple-webapp
        image: simple-webapp
```

### Taint and Tolerations
- Node: Taints, no unwanted pods to be scheduled on the node
- Pods: Tolerations, allow pods to be scheduled on nodes with matching taints
- Taints and tolerations work together to ensure that pods are not scheduled onto inappropriate nodes.
- Taint and Toleration doesn't tell where the pods should be scheduled, it only tells where the pods should not be scheduled. 
- If a node has a taint, it means that no pods can be scheduled on that node unless they have a matching toleration. If a pod has a toleration, it means that it can be scheduled on nodes with matching taints.

#### Command
- Taint a node: `kubectl taint nodes <node-name> <key>=<value>:<taint-effect>`, taint-effect can be NoSchedule, PreferNoSchedule, or NoExecute
  - NoExecute: Taints the node so that no pods can be scheduled on it, and any existing pods on the node will be evicted, unless pod has matching teleration
  - NoSchedule: Taints the node so that no pods can be scheduled on it, but existing pods will not be evicted
  - PreferNoSchedule: Taints the node so that the scheduler will try to avoid schedulin
- Untaint a node, append dash: `kubectl taint nodes <node-name> <key>:<taint-effect>-`
- Add toleration is below
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: nginx-container
      image: nginx
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"

```

### Node Selectors
- Allows us to say which nodes we want to schedule our pods on

#### Command
- Add label to a node: `kubectl label nodes <node-name> <key>=<value>`
- More complex with Node Affinity
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: data-processor
      image: data-processor
  nodeSelector:
    size: large
```

### Node Affinity
- More advance expression of node selector, allows us to specify rules for scheduling pods on nodes based on labels and conditions
- Below works the same as above
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: data-processor
      image: data-processor
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution: 
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In                  # operator can be In, NotIn, Exists, DoesNotExist, Gt, Lt
            values:
            - large
```

#### Affinity types
- RequiredDuringSchedulingIgnoredDuringExecution: If the rules are not met, the pod will not be scheduled. If the rules are violated after the pod is scheduled, it will not be evicted.
- PreferredDuringSchedulingIgnoredDuringExecution: if the rules are not met, the pod will still be scheduled, but it will be less preferred. If the rules are violated after the pod is scheduled, it will not be evicted.
- RequiredDuringSchedulingRequiredDuringExecution: If the rules are not met, the pod will not be scheduled. If the rules are violated after the pod is scheduled, it will be evicted.

### Affinity vs Taint & Tolerations
- Taints and tolerations does not tell where the pods should be scheduled, it only tells where the pods should not be scheduled.
- Affinity tells where the pods should be scheduled, however, it does not guarantee that other pods will not be scheduled on the same node
- Combination of both can be used to achieve more complex scheduling requirements
  - TnT to prevent other pods form being placed on our nodes
  - Affinity to specify where our pods should be placed, and prevent our pods from being placed on other nodes

### Resource Limits
- kube-scheduler checks resources before scheduling a pod to a node
- CPU cannot use resources more than its limit, but Memory can use more than its limit (but will get OOM killed if constantly)
- By default, no resource limit are set. A pod can use all the resources of a node, which can lead to resource contention and instability in the cluster.

#### CPU behavior
- No request, no limit: pod can use all CPU on the node, but is only guaranteed a minimal fair share if multiple pods compete. CPU can be throttled during contention.
- No request, limit: pod can use all the CPU resources of the node, but it will be throttled if it tries to use more than its limit
- Request, limit: pod is guaranteed to have the requested CPU resources, but it will be throttled if it tries to use more than its limit
- Request, no limit: pod is guaranteed to have the requested CPU resources, but it can use all the CPU resources of the node if needed, which can lead to resource contention and instability in the cluster

#### Memory
- No request, no limit: Pod can use all memory on the node; may cause resource contention and instability if multiple pods use lots of memory.
- No request, limit: Pod can use memory freely until it hits the limit; it will be OOMKilled if it tries to exceed the limit.
- Request, limit: Pod is guaranteed the requested memory; will be OOMKilled if it exceeds the limit.
- Request, no limit: Pod is guaranteed the requested memory, but can use more if available; Kubernetes cannot throttle memory, so pod may still be OOMKilled if node runs out of memory.

#### Limit range

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-resource-constraints
spec:
  limits:
  - type: Container
    default:
      cpu: 500m               # or memory: 512Mi
    defaultRequest:
      cpu: 200m                # or memory: 512Mi
    max:
      cpu: "1"
    min:
      cpu: 100m
```

#### Resource Quotas
- Create quota at namespace level to limit the total resource consumption of all pods in the namespace, which can help to prevent a single team or application from consuming all the resources in the cluster and ensure fair resource allocation among different teams and applications
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-resource-quota
spec:
  hard:
    requests.cpu: "4"           # total CPU request for all pods in the namespace cannot exceed 4
    request.memory: 8Gi                # total memory request for all pods in the namespace cannot exceed 8Gi
    limits.cpu: "10"              # total CPU limit for all pods in the namespace cannot exceed 10
    limits.memory: 16Gi             # total memory limit for all pods in the namespace cannot exceed 16Gi
```

### DaemonSets
- Deploy 1 pod per node, used for cluster-wide services like log collection, monitoring, etc.
- Kube-proxy is deployed as a DaemonSet, which ensures that there is a kube-proxy pod running on each node in the cluster

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: kube-proxy
  name: kube-proxy
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: kube-proxy
  template:
    metadata:
      labels:
        name: kube-proxy
    spec:
      containers:
      - name: kube-proxy
        image: k8s.gcr.io/kube-proxy:v1.13.0
```

#### Commands
- Get DaemonSet: `kubectl get daemonset`
- Describe DaemonSet: `kubectl describe daemonset <daemonset-name>`

### Static Pods
- Static pods are managed directly by the kubelet on a specific node, without the involvement of the kube-apiserver
- Only works for pods, not for other resources like services, deployments, etc.
- When kubelet creates a static pod, a mirror pod is created in the kube-apiserver with the same name and namespace, but with a different UID. This allows the kube-apiserver to track the status of the static pod and report it to the user, while the kubelet continues to manage the lifecycle of the static pod independently.
- Use case: deploy control plane component as pod on a node

| Static pod                                        | daemonset                             |
|---------------------------------------------------|---------------------------------------|
| Created by kubelet, not managed by kube-apiserver | Created and managed by kube-apiserver | 
| Deploy Control Plane components as static pods    | Deploy monitoring agents, log collectors, etc. as daemonsets |
| Ignored by kubescheduler | Ignored by kubescheduler |

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: static-busybox
  name: static-busybox
spec:
  containers:
  - command:
    - sleep
    - "1000"
    image: busybox:1.28.4
    name: static-busybox
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
```

#### Commands
- Get static pods: `kubectl get pods `, and look for pods with `-<node-name>` appended in the name
- Create static pod: `kubectl run my-static-pod --image=nginx --dry-run=client -o yaml > /etc/kubernetes/manifests/my-static-pod.yaml`
- To delete static pod
  - identify the node
  - ssh into the node
  - run `ps -ef |  grep /usr/bin/kubelet `
  - get the path of the static pod manifest file, which is usually `grep -i staticpod /var/lib/kubelet/config.yaml`
  - delete the static pod manifest file, which will cause the kubelet to delete the static pod


### Priority Classes

### Multiple Schedulers

### Configuring Scheduler profiles

### Admission Controllers

### Validating and Mutating Admission Controllers



## 4. Observability
## 5. Pod Design
## 7. State Persistence
## 8. Security
## 8. Storage
## 6. Networking

## 9. Exam Tips & Speed Tricks

### Aliases
```bashrc
# --- Core ---
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias krm='kubectl delete'

# --- Apply / Create ---
alias ka='kubectl apply -f'
alias kcf='kubectl create -f'
alias kc='kubectl create'
alias kr='kubectl run'

# --- Logs / Exec ---
alias kl='kubectl logs'
alias ke='kubectl exec -it'

# --- Namespaces ---
alias kn='kubectl config set-context --current --namespace'

# --- YAML generation ---
export do='--dry-run=client -o yaml'
```

### Command Tips
- Create NGINX pod: `kubectl run nginx --image=nginx`
- Dry run to generate pod YAML: `kubectl run nginx --image=nginx --dry-run=client -o yaml`
- Crete deployment: `kubectl create deployment nginx --image=nginx`
- Dry run to generate deployment YAML and save: `kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > nginx-deployment.yaml`
- Make changes to the YAML file and create: `kubectl create -f nginx-deployment.yaml`

### Imperative Command Tips
- `--dry-run`: By default as soon as the command is run, the resource will be created. 
- `--dry-run=client` If you simply want to test your command . This will not create the resource, instead, tell you whether the resource can be created and if your command is right.
- `-o yaml`: This will output the resource definition in YAML format on screen
- `kubectl api-resources | grep -E 'nodes|pods|services|deployments'` to list all the resources we can create with kubectl

#### Pod
- `kubectl run nginx --image=nginx`
-  `kubectl run redis -l tier=db --image=redis:alpine`, with label `tier=db`
- `kubectl run nginx --image=nginx --dry-run=client -o yaml`, 

#### Deployment
- `kubectl create deployment --image=nginx nginx`
- `kubectl create deployment --image=nginx nginx --dry-run=client -o yaml`
- `kubectl create deployment nginx --image=nginx --replicas=4`
- `kubectl scale deployment nginx --replicas=4`
- `kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > nginx-deployment.yaml`, save file and modify

#### Service
Create a Service named redis-service of type ClusterIP to expose pod redis on port 6379
- `kubectl expose pod redis  --type=ClusterIP --port=6379 --name redis-service --dry-run=client -o yaml` (This will automatically use the pod's labels as selectors)
- `kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml` (This will not use the pods labels as selectors, instead it will assume selectors as app=redis)
Create a Service named nginx of type NodePort to expose pod nginx's port 80 on port 30080 on the nodes:
- `kubectl expose pod nginx --type=NodePort --port=80 --name=nginx-service --dry-run=client -o yaml` (This will automatically use the pod's labels as selectors, but you cannot specify the node port. You have to generate a definition file and then add the node port in manually before creating the service with the pod.)
- `kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml` (This will not use the pods labels as selectors)











