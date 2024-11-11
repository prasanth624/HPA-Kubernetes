# Horizontal Pod Autoscaling (HPA) in Kubernetes⚖️
🚀Imagine you are managing an online store, and you’ve deployed your store’s website in a Kubernetes cluster. The website is built using multiple pods, which handle different parts of the website like displaying products, handling user accounts, processing orders, etc.

You want to ensure that your website can handle sudden traffic spikes (e.g., during a big sale) without crashing, but you also don’t want to waste resources by running too many pods when traffic is low.

This is where Horizontal Pod Autoscaling (HPA) comes in.

## What is Horizontal Pod Autoscaling (HPA)?🤔

HPA (Horizontal Pod Autoscaler) is a Kubernetes feature that monitors the resource usage (such as CPU or memory) of your pods and automatically adjusts the number of replicas based on the demand.
- When the application experiences increased traffic (such as during a flash sale), Horizontal Pod Autoscaler (HPA) scales up the number of pods (or replicas) in the deployment to handle the increased load.
- When traffic decreases (such as after the sale ends), HPA scales down the number of pods to reduce resource consumption and optimize costs.

### Problem Without HPA

Let’s say you have a fixed number of pods running your website — let’s say 2 pods. If there’s a sudden surge in traffic (like on a busy shopping day), those 2 pods may not be enough to handle all the visitors, leading to slow performance or website crashes. On the other hand, if the traffic is very low (after the sale), you might end up running more pods than needed, wasting resources.

### How HPA Solves This Problem?💡

HPA solves this by automatically scaling the number of pods up or down based on how much workload the pods are handling at any given time.

### Key Concepts of HPA

1. **Target Metrics**: 
   HPA uses resource metrics (like CPU or memory usage) to decide when to scale. 
   - For example, if the average CPU usage exceeds 80%, HPA may decide to add more pods.
   - And, if CPU usage drops below 30%, it may reduce the number of pods.

2. **Min and Max Replicas**:
   — Min Replicas: The minimum number of pods to ensure the website always has enough capacity. Let’s say you set this to 2 pods.
   — Max Replicas: The maximum number of pods to prevent the system from scaling out of control. Let’s say you set this to 10 pods.

3. **Metrics Server**: 
   For HPA to work, Kubernetes requires a metrics server to collect data on pod resources like CPU and memory. This data is used by HPA to make scaling decisions.


## Let's do some Hands-On🧑‍💻
### Prerequisites

- Kubernetes Cluster (I have used Minikube)
- kubectl


## Steps

### Step 1: Create a Deployment

Create a file called `deployment.yml` and add the following code:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: green-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
      version: green
  template:
    metadata:
      labels:
        app: my-app
        version: green
    spec:
      containers:
      - name: my-app
        image: itsprasanth4/nginx:green
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 10m
          requests:
            cpu: 2m
```

Apply the deployment using:

```bash
kubectl apply -f deployment.yml
```

Verify the deployment:

```bash
kubectl get deployment,pods
```

You should see the pods running:

```
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/green-app   2/2     2            2           33s

NAME                             READY   STATUS    RESTARTS   AGE
pod/green-app-76f6bff8fd-xct6d   1/1     Running   0          33s
pod/green-app-76f6bff8fd-xwhp5   1/1     Running   0          33s
```

### Step 2: Create a Service to Route Traffic

Create a file called `service.yml` with the following content:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
    version: green
  ports:
  - port: 80
    targetPort: 80
```

Apply the service:

```bash
kubectl apply -f service.yml
```

Verify the service:

```bash
kubectl get svc my-app
```

You should see:

```
NAME     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
my-app   ClusterIP   10.99.103.248   <none>        80/TCP    46s
```

### Step 3: Enable Metrics Server

Ensure the **metrics server** is enabled to provide resource usage data for HPA:

```bash
kubectl top pods
error: Metrics API not available
```

If the metrics API is not available, enable the metrics server:

For Minikube:

```bash
minikube addons enable metrics-server
```

For other Kubernetes clusters(one master node cluster)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

After enabling:

```bash
kubectl top pods
```
```bash
NAME                         CPU(cores)   MEMORY(bytes)
green-app-76f6bff8fd-xct6d   0m           2Mi
green-app-76f6bff8fd-xwhp5   0m           2Mi
```

### Step 4: Create HPA Object📈

Create a file called `hpa.yml` with the following content:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: green-app
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: green-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Apply the HPA:

```bash
kubectl apply -f hpa.yml
```

Verify the HPA:

```bash
kubectl get hpa
```
```bash
NAME        REFERENCE              TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
green-app   Deployment/green-app   cpu: 0%/50%   2         10        2          17s
```

### Step 5: Test the Autoscaling
lets increase the load on pods
Open another terminal and run the below command

```bash
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://my-app; done"
```
Now go back to previous terminal and apply below command

```bash
kubectl get hpa -w
```
```bash
NAME        REFERENCE              TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
green-app   Deployment/green-app   cpu: 0%/50%   2         10        2          6m33s
green-app   Deployment/green-app   cpu: 150%/50%   2         10        2          6m45s
green-app   Deployment/green-app   cpu: 150%/50%   2         10        4          7m
green-app   Deployment/green-app   cpu: 150%/50%   2         10        6          7m15s
green-app   Deployment/green-app   cpu: 200%/50%   2         10        6          7m45s
green-app   Deployment/green-app   cpu: 200%/50%   2         10        8          8m
green-app   Deployment/green-app   cpu: 100%/50%   2         10        8          8m46s
green-app   Deployment/green-app   cpu: 100%/50%   2         10        10         9m1s
```

You will see HPA increasing the number of pods based on CPU utilization.

### Step 7: Verify the Pods

Check the scaling of your pods:

```bash
kubectl get pods,deployments
```
```bash
NAME                             READY   STATUS    RESTARTS   AGE
pod/green-app-76f6bff8fd-bnrqg   1/1     Running   0          3m34s
pod/green-app-76f6bff8fd-ckhqn   1/1     Running   0          3m49s
pod/green-app-76f6bff8fd-jm6c7   1/1     Running   0          3m34s
pod/green-app-76f6bff8fd-mwph5   1/1     Running   0          108s
pod/green-app-76f6bff8fd-nrvr7   1/1     Running   0          2m49s
pod/green-app-76f6bff8fd-qs2sj   1/1     Running   0          108s
pod/green-app-76f6bff8fd-v2swc   1/1     Running   0          2m49s
pod/green-app-76f6bff8fd-xct6d   1/1     Running   0          19m
pod/green-app-76f6bff8fd-xwhp5   1/1     Running   0          19m
pod/green-app-76f6bff8fd-zj2sn   1/1     Running   0          3m49s
pod/load-generator               1/1     Running   0          4m25s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/green-app   10/10   10           10          19m
```

Once the load generator is stopped (by pressing Ctrl+C in the terminal where it was running), the load on the pods will decrease, and the HPA will automatically adjust by scaling down the pods accordingly.
You can verify as well

```bash
kubectl get pods,deployment
```
```bash
NAME                             READY   STATUS    RESTARTS   AGE
pod/green-app-76f6bff8fd-xct6d   1/1     Running   0          27m
pod/green-app-76f6bff8fd-xwhp5   1/1     Running   0          27m
pod/load-generator               0/1     Error     0          13m

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/green-app   2/2     2            2           27m
```

## Clean Up

Clean up the resources
```bash
kubectl delete -f /path/to/files/directory
```
## Conclusion

In simple terms, Horizontal Pod Autoscaling (HPA) in Kubernetes is like a smart system that automatically adjusts the number of pods running your application based on demand. When workload increases, it adds more pods to handle the load, and when workload decreases, it reduces the number of pods to save resources. This helps ensure your application is always ready for any traffic spikes while optimizing the use of resources, making it cost-efficient and scalable.

## Contributing

If you have ideas for improvements or new features, feel free to submit a pull request. 🤝

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details. 📜
