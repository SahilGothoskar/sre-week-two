# Task 2: Troubleshoot a Production Incident, Find the Root Cause, Resolve the Incident & Create Action Items to Mitigate It

## Scenario Introduction:

Your teammate at UpCommerce has just pushed an update after some changes made to the site's code by the development team. Ever since that push, you have been getting pager alerts that the UpCommerce.com service is down. As SRE team lead, it is your job to handle and resolve problems that happen in the UpCommerce Kubernetes cluster. You will also need to keep your Incident Commander (IC) and Communications Lead (CL) up to date so they can manage UpCommerce's users.


## Troubleshooting:

Below command shows the overall view of deployment sre in our minikube cluster.

```bash
kubectl get deploy -n sre
```


NOTE: To deploy the code please use this repo


## Troubleshooting:

Below command shows the overall view of deployment sre in our minikube cluster.

```bash
kubectl get deploy -n sre
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
grafana                             1/1     1            1           37m
prometheus-kube-state-metrics       1/1     1            1           39m
prometheus-prometheus-pushgateway   1/1     1            1           39m
prometheus-server                   1/1     1            1           39m
upcommerce-app-two                  0/1     1            0           15m
```

Now lisiting the pods in the sre namespace 

```bash
$ kubectl get pods -n sre
NAME                                                READY   STATUS    RESTARTS   AGE
grafana-557d966c8c-wlm78                            1/1     Running   0          36m
prometheus-alertmanager-0                           1/1     Running   0          38m
prometheus-kube-state-metrics-65468947fb-b6j5b      1/1     Running   0          38m
prometheus-prometheus-node-exporter-nzt48           1/1     Running   0          38m
prometheus-prometheus-pushgateway-76976dc66-chmk7   1/1     Running   0          38m
prometheus-server-8444b5b7f7-xp88x                  2/2     Running   0          38m
upcommerce-app-two-56cff9c64d-tznnl                 0/1     Pending   0          13m
```

Now we need to describe the failing pods to figure out the issue. 

```bash
$ kubectl describe pods upcommerce-app-two-56cff9c64d-tznnl -n sre
Name:             upcommerce-app-two-56cff9c64d-tznnl
Namespace:        sre
Priority:         0
Service Account:  default
Node:             <none>
Labels:           app=upcommerce-app-two
                  pod-template-hash=56cff9c64d
Annotations:      <none>
Status:           Pending
IP:               
IPs:              <none>
Controlled By:    ReplicaSet/upcommerce-app-two-56cff9c64d
Containers:
  upcommerce:
    Image:      uonyeka/upcommerce:v3
    Port:       5000/TCP
    Host Port:  0/TCP
    Limits:
      cpu:     10
      memory:  4Gi
    Requests:
      cpu:        10
      memory:     4Gi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-qc8p6 (ro)
Conditions:
  Type           Status
  PodScheduled   False 
Volumes:
  kube-api-access-qc8p6:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Guaranteed
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  4m41s  default-scheduler  0/1 nodes are available: 1 Insufficient cpu. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod..
```

## Root Cause:

The problem being the node has insufficient cpu.

```
Warning  FailedScheduling  4m41s  default-scheduler  0/1 nodes are available: 1 Insufficient cpu. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod..
```

The maximium cpu which can be allocated to the pod can be checked:

```bash
 $ kubectl get nodes minikube  -o yaml | grep -A2 alloc
  allocatable:
    cpu: "2"
    ephemeral-storage: 32847680Ki
```

The current version of the deployment manifest/yaml file needs to be  checked to get actual configs.

```bash
$ kubectl describe deploy upcommerce-app-two -n sre
Name:                   upcommerce-app-two
Namespace:              sre
CreationTimestamp:      Fri, 12 Apr 2024 16:04:04 +0000
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=upcommerce-app-two
Replicas:               1 desired | 1 updated | 1 total | 0 available | 1 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=upcommerce-app-two
  Containers:
   upcommerce:
    Image:      uonyeka/upcommerce:v3
    Port:       5000/TCP
    Host Port:  0/TCP
    Limits:
      cpu:        10
      memory:     4Gi
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      False   MinimumReplicasUnavailable
  Progressing    False   ProgressDeadlineExceeded
OldReplicaSets:  <none>
NewReplicaSet:   upcommerce-app-two-56cff9c64d (1/1 replicas created)
```

If we check the above deployment we can see the deployment is using 10 cpu and maximum allocated is 2. So in order to fix the deployment we need to reduce the cpu.

Curent code of the deployment:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: upcommerce-app-two
spec:
  replicas: 1
  selector:
    matchLabels:
      app: upcommerce-app-two
  template:
    metadata:
      labels:
        app: upcommerce-app-two
    spec:
      containers:
        - name: upcommerce
          image: uonyeka/upcommerce:v3
          ports:
            - containerPort: 5000
          imagePullPolicy: Always
          resources:
            limits:
              cpu: "10"
              memory: "4Gi"
```

Updated code with CPU reduced to 1

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: upcommerce-app-two
spec:
  replicas: 1
  selector:
    matchLabels:
      app: upcommerce-app-two
  template:
    metadata:
      labels:
        app: upcommerce-app-two
    spec:
      containers:
        - name: upcommerce
          image: uonyeka/upcommerce:v3
          ports:
            - containerPort: 5000
          imagePullPolicy: Always
          resources:
            limits:
              cpu: "1"
              memory: "4Gi"
```

Fixed Issue

```bash
kubectl get pods -n sre
NAME                                                READY   STATUS    RESTARTS   AGE
grafana-557d966c8c-wlm78                            1/1     Running   0          58m
prometheus-alertmanager-0                           1/1     Running   0          60m
prometheus-kube-state-metrics-65468947fb-b6j5b      1/1     Running   0          60m
prometheus-prometheus-node-exporter-nzt48           1/1     Running   0          60m
prometheus-prometheus-pushgateway-76976dc66-chmk7   1/1     Running   0          60m
prometheus-server-8444b5b7f7-xp88x                  2/2     Running   0          60m
upcommerce-app-two-846fd54f5d-jps8v                 1/1     Running   0          26s
```


