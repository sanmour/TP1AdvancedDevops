## 0. Technologies used : 
![[TP1 GitOps-20241007213155800.webp]]

### 1. Setting up the necessary configurations:

Our app is a simple Flask API that responds with a JSON message that includes a greeting and the current time.

1. First we included all the requirements in `requirements.txt`
3. We created a Dockerfile for Containerization

```Dockerfile
FROM python:3.8-slim
WORKDIR /app
COPY . /app
RUN pip install -r requirements.txt
EXPOSE 5000
CMD ["python", "app.py"]
```

4. We built and Pushed the  Docker Image using the following command :

```bash
docker build -t sandramourali/helloworld-flask .
```
![[TP1 GitOps-20241007203719218.webp]]
```yaml
docker push sandramourali/helloworld-flask
```
flowchart TD
Start --> Stop
![[TP1 GitOps-20241007203827449.webp]]
5. We set up Kubernetes with Minikube
```bash
minikube start
```

6. We created Kubernetes YAML Files
##### `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-container
        image: sandramourali/helloworld-flask:latest
        ports:
        - containerPort: 5000
```

##### `service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  selector:
    app: flask-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
  type: LoadBalancer
```

## 2.Setting up Minikube and Kubernetes

We applied the YAML files we created to the Kubernetes Cluster :
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```
To test whether our configurations were correctly applied we used:
```Shell
kubectl get deployments
kubectl get pods
kubectl get services
```
![[TP1 GitOps-20241007204150189.webp]]
## 3.GitOps with ArgoCD
ArgoCD is one of the best tools to automate deployments as everytime we push changes to the repository, ArgoCD will automatically deploy the new version of our Flask app.
1. First we installed and set up ArgoCd :
```shell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
![[TP1 GitOps-20241007203923291.webp]]
3. Then we created and configured our project via ArgoCD's UI :
![[TP1 GitOps-20241007203611484.webp]]
2. Once the configurations done, we synced up with our Github Repository :
![[TP1 GitOps-20241007203639386.webp]]

## 4. Monitoring and alerts:
1. For the monitoring part, ArgoCD already does the monitoring as it provides basic health status indicators for applications. It shows whether an application is Healthy, Progressing, Degraded, or Suspended. This information can be viewed in the Argo CD UI.
![[TP1 GitOps-20241007205022958.webp]]

2. For the alerts part, we integrated Prometheus with ArgoCd:
	We created an alerts pipeline with Prometheus by setting up the following `alert.yaml` config:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: example-alert-rules
  namespace: monitoring
spec:
  groups:
    - name: example
      rules:
        - alert: HighCPUUsage
          expr: avg(rate(container_cpu_usage_seconds_total{job="kubelet"}[5m])) by (instance) > 0.5
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High CPU Usage on {{ $labels.instance }}"
            description: "CPU usage is above 50% for more than 5 minutes."

```
We applied the alert using the following command :
```Shell
kubectl apply -f alert.yaml
```
Then we restarted the deployment to apply changes made :
```shell
kubectl rollout restart deployment prometheus-kube-prometheus-alertmanager -n monitoring
```
Afterwards we synced with ArgoCD
```shell
argocd app sync prometheus-app
```

![[TP1 GitOps-20241007211557759.webp]]
