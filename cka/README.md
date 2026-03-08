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
- Non-namespaced objects
- Priorities: range of number (apps): -2147483648 and 10000000000
- Range for internal (system): 1000 000 000 to 2 0000 000 000
- Global value, single priority class can be used by multiple pods, and a pod can only have one priority class

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000000
description: "blabla"
preemptionPolicy: PreemptLowerPriority # can be PreemptLowerPriority (Kill existing low priority workload) or Never (cannot pre-empt other existing pods), default is PreemptLowerPriority
```

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
      ports:
        - containerPort: 8080
  priorityClassName: high-priority
 ```

#### Command
- List priority classes: `kubectl get priorityclass`
- Get pods priority colum: `kubectl get pods -o custom-columns="NAME:.metadata.name,PRIORITY:.spec.priorityClassName"`

### Multiple Schedulers
- Custom schedulers can be deployed, can have multiple schedulers at a time

#### Command
- View scchedulers: `kubectl get pods -n kube-system`
- View scheduler logs: `kubectl logs <scheduler-pod-name> -n kube-system`
- Deploy custom scheduler: `wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-scheduler`
- Deploy additional scheduler as a pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end        
spec:
  containers:
  - command:
    - kube-scheduler
    - --address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf- --config=etc/kubernetes/scheduler-config.yaml
    image: k8s.gcr.io/kube-scheduler:v1.13.0
    name: my-scheduler
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
  schedulerName: my-scheduler
```

### Configuring Scheduler profiles
- **Four pipeline stages in order:** Queue Sort → Filter → Score → Bind
- **Each stage uses plugins** that can be enabled or disabled to customize scheduling behavior
- **Multiple profiles can run in a single scheduler binary**, each with a different
   name and plugin configuration
- **Pods select a profile** via `schedulerName` in their spec, defaulting to `default-scheduler` if not specified
- **Multiple profiles prevent race conditions** that occur when running separate scheduler binaries competing for the same node

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
    plugins:
      score:
        disabled:
          - name: NodeResourcesBalancedAllocation
        enabled:
          - name: MyCustomScorePlugin
  - schedulerName: high-priority-scheduler
    plugins:
      filter:
        disabled:
          - name: TaintToleration
```

### Admission Controllers
- Kubelet -> authentication -> authorization -> admission-controllers -> create pod
- For context: 
  - Authentication: certificate, token, etc.
  - Authorization: RBAC, etc
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create", "get", "list", "watch", "update", "patch", "delete"]
    resourceNames: [""] # optional, if specified, only allow access to the specified resources
```
- Admission controllers helps us implement better security, resource management, and other policies in the cluster by intercepting requests to the kube-apiserver and modifying or rejecting them based on predefined rules.
- AC
  - AlwaysPullImages: Always pull images, even if they are already present on the node
  - DefaultStorageClass: Automatically set the default storage class for persistent volume claims that do not specify a storage class
  - NamespaceExists: Reject requests to create or update objects in namespaces that do not exist
  - NameSpaceAutoProvision: Automatically create namespaces when they are referenced in object creation requests but do not exist

#### command
- View enabled admission controllers: `kube-apiserver -h | grep enable-admission-plugins`
- Add admission controller to kube-apiserver.yaml: `- --enable-admission-plugins=NamespaceAutoProvision,DefaultStorageClass`
- Add webhoook admission controller: `- --enable-admission-plugins=ImagePolicyWebhook`, ``- --admission-control-config-file=/etc/kubernetes/imgvalidation/admission-configuration.yaml``

### Validating and Mutating Admission Controllers
- Mutating admission controller can modify the request object before it is persisted
- Validating admission controller can be used to enforce policies and security in the cluster, to reject or allow certain requests based on predefined rules
- Ex:
  - Mutating
    - `NamespaceAutoProvision`: Automatically create namespaces when they are referenced in object creation requests but do not exist
  - Validating
    - `NamespaceExists`: Reject requests to create or update objects in namespaces that do not exist
- Two special admission controllers for custom policies:
  - `ValidatingAdmissionWebhook`: Allows you to define custom validation logic in an external webhook service. When a request is made to the kube-apiserver, it sends the request data to the webhook service, which can then validate the request and return a response indicating whether the request should be allowed or rejected.
  - `MutatingAdmissionWebhook`: Similar to the validating webhook, but allows you to modify the request object before it is persisted. This can be used to automatically add labels, annotations, or other fields to objects based on certain criteria.
- How to setup:
  1. Create a admission webhook service that implements the desired validation or mutation logic
     - `/validate` for validating webhook, 
     - `/mutate` for mutating webhook
  2. Configure webhook on kubernetes by creating webhook configuration objects (ValidatingWebhookConfiguration or MutatingWebhookConfiguration) that specify the webhook service and the rules for when the webhook should be invoked

