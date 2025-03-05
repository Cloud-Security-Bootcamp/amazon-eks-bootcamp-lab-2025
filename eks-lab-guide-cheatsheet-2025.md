# Amazon EKS CheatSheet (with AWS CloudShell)

## Cluster Setup (Inside AWS CloudShell)

Connect to cluster:

```
aws eks update-kubeconfig --region us-east-1 --name eks-lab-cluster
```

Verify nodes:

```
kubectl get nodes
```

# Lab 1 - Vulnerable Java Pod

**Create and apply:**

```
cat <<EOF > vulnerable-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: vulnerable-java-app
  labels:
    app: vulnerable-java
spec:
  containers:
  - name: vulnerable-java-container
    image: ghcr.io/datadog/vulnerable-java-application:latest
    ports:
    - containerPort: 8080
apiVersion: v1
kind: Service
metadata:
  name: vulnerable-java-service
spec:
  selector:
app: vulnerable-java
  ports:
protocol: TCP
port: 80
targetPort: 8080
  type: LoadBalancer
EOF
```

Deploy app:

```
kubectl apply -f vulnerable-app.yaml
```

**Verify:**

```
kubectl get pods
kubectl get services
```

Test:

```
curl http://<EXTERNAL-IP>
```

Cleanup:

```
kubectl delete -f vulnerable-app.yaml
```

## Troubleshooting

Check pods:

```
kubectl get pods
kubectl describe pod vulnerable-java-app
```

Check logs:

```
kubectl logs vulnerable-java-app
```

Test service:

```
kubectl run -it --rm debug --image=busybox -- sh
```

Once inside the pod

```
wget -O- http://vulnerable-java-service:80
```

Fix debug pod:

```
kubectl delete pod debug --force --grace-period=0
```

---

# Lab 2 - Dev & Prod Environments

** Create namespaces: **

```
kubectl create namespace dev test prod
```

** Deploy dev (vulnerable app):**

```
cat <<EOF > dev-vulnerable-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vulnerable-java-app
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vulnerable-java
  template:
    metadata:
      labels:
        app: vulnerable-java
    spec:
      containers:
      - name: vulnerable-java-container
        image: ghcr.io/datadog/vulnerable-java-application:latest
        ports:
        - containerPort: 8080
apiVersion: v1
kind: Service
metadata:
  name: vulnerable-java-service
  namespace: dev
spec:
  selector:
app: vulnerable-java
  ports:
port: 80
targetPort: 8080
  type: LoadBalancer
EOF
```

** Deploy app (using Deployment with 2 replicas):**

```
kubectl apply -f dev-vulnerable-app-deployment.yaml
```

**Deploy prod (NGINX):**

```
cat <<EOF > prod-nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: prod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: prod
spec:
  selector:
app: nginx
  ports:
port: 80
targetPort: 80
  type: LoadBalancer
EOF
```

** Deploy service:**

```
kubectl apply -f prod-nginx-deployment.yaml
```

** Verify:**

```
kubectl get deployments,pods,services -n dev
kubectl get deployments,pods,services -n prod
```

** Test: **

```
curl http://<dev-EXTERNAL-IP>
curl http://<prod-EXTERNAL-IP>
```

** Scale prod: **

```
kubectl scale deployment nginx-app -n prod --replicas=3
```

** Test resilience: **

```
kubectl delete pod <nginx-pod-name> -n prod
kubectl get pods -n prod
```

** Cleanup: **

```
kubectl delete -f dev-vulnerable-app-deployment.yaml
kubectl delete -f prod-nginx-deployment.yaml
kubectl delete namespace dev test prod
```

## Troubleshooting

** Check pods: **

```
kubectl get pods [-n <namespace>]
kubectl describe pod <pod-name> [-n <namespace>]
```

** Check logs: **

```
kubectl logs <pod-name> [-n <namespace>]
```

** Test service: **

```
kubectl run -it --rm debug --image=busybox -- sh
```

Once you are in the pod

```
wget -O- http://<service-name>.<namespace>:80
```

** Fix debug pod: **

```
kubectl delete pod debug --force --grace-period=0
```


This cheat sheet is streamlined for Lab 1 & Lab 2,  It’s a quick reference for setting up, running, and debugging the vulnerable Java pod in EKS.
