# ToDo Application Kubernetes Deployment

Check if the namespace exists:

```bash
kubectl get namespaces
```

## Deploy the application

Apply the Deployment manifest:

```bash
kubectl apply -f .infrastructure/deployment.yml
```

Apply the Horizontal Pod Autoscaler:

```bash
kubectl apply -f .infrastructure/hpa.yml
```

Check the Deployment:

```bash
kubectl get deployments 
```

Check application pods:

```bash
kubectl get pods 
```
Check the Horizontal Pod Autoscaler:

```bash
kubectl get hpa 
```

Detailed HPA information can be displayed with:

```bash
kubectl describe hpa todoapp-hpa 
```

## Resource requests and limits

Each application container has the following resource configuration:
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

The CPU request of `100m` reserves 0.1 CPU core for each pod. This amount is
sufficient for the ToDo application while it is idle and provides a useful
baseline for CPU-based autoscaling. The memory request of `128Mi` provides enough memory for the lightweight
Django application under normal load. The CPU limit of `500m` allows a pod to use up to half of one CPU core during
temporary load spikes while preventing a single container from consuming
excessive CPU resources. The memory limit of `256Mi` allows the application to temporarily consume more
memory than its request while still protecting the Kubernetes node from
uncontrolled memory consumption.  The Deployment uses two replicas, so two application pods are running in the
idle state.

## RollingUpdate strategy

The Deployment uses the following update strategy:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

`maxUnavailable: 0` ensures that the existing application pods remain
available while a new version is being deployed. Kubernetes waits until a new
pod becomes ready before removing an old pod.

`maxSurge: 1` allows Kubernetes to create one additional pod during an update.

For a deployment with two replicas, this means that a maximum of three pods
can temporarily exist during an update.

This strategy was selected to avoid application downtime while limiting the
amount of additional resources required during deployment.

## Horizontal Pod Autoscaler

The HPA configuration allows the number of application pods to vary between
2 and 5.

```yaml
minReplicas: 2
maxReplicas: 5
```

Two replicas provide application availability during normal and idle
operation.

Five replicas provide additional capacity during increased load without
allowing uncontrolled scaling.

The application is scaled based on both CPU and memory usage.

CPU target:

```yaml
averageUtilization: 70
```

Memory target:

```yaml
averageUtilization: 75
```

A CPU target of 70% leaves additional CPU capacity in each pod before scaling
is required.

A memory target of 75% allows normal memory usage while creating additional
pods before the containers approach their configured memory limits.

HPA calculates resource utilization relative to the resource requests
configured in the Deployment.


## Access the application

The application can be accessed locally using port-forward.

Forward local port `8000` to the Deployment:

```bash
kubectl port-forward -n mateapp deployment/todoapp 8000:8000
```
