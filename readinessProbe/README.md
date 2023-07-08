## Part 4 - readinessProbe

- The kubelet uses readiness probes to know when a container is ready to start accepting traffic. A Pod is considered ready when all of its containers are ready. One use of this signal is to control which Pods are used as backends for Services. When a Pod is not ready, it is removed from Service load balancers.

- Sometimes, applications are temporarily unable to serve traffic. For example, an application might need to load large data or configuration files during startup, or depend on external services after startup. In such cases, you don't want to kill the application, but you don't want to send it requests either. Kubernetes provides readiness probes to detect and mitigate these situations. A pod with containers reporting that they are not ready does not receive traffic through Kubernetes Services.

> **Note:** Readiness probes runs on the container during its whole lifecycle.

> **Caution:** Liveness probes do not wait for readiness probes to succeed. If you want to wait before executing a liveness probe you should use initialDelaySeconds or a startupProbe.

- Readiness probes are configured similarly to liveness probes. The only difference is that you use the readinessProbe field instead of the livenessProbe field.

- Configuration for HTTP and TCP readiness probes also remains identical to liveness probes.

- Readiness and liveness probes can be used in parallel for the same container. Using both can ensure that traffic does not reach a container that is not ready for it, and that containers are restarted when they fail.

- Create a `http-readiness.yaml` and input text below.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness
spec:
  replicas: 3
  selector:
    matchLabels:
      test: readiness
  template:
    metadata:
      labels:
        test: readiness
    spec:
      containers:
      - name: readiness
        image: clarusway/readinessprobe
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 3
          periodSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-http
spec:
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30001
  selector:
    test: readiness
  type: NodePort
```

- In this image (clarusway/readinessprobe), for the `first 45 seconds` the container returns a status of 500. Than the container will return a status of `200`. 

```py
app = Flask(__name__)

start = time.time()

@app.route('/')
def home():
    return "Welcome to Clarusway Kubernetes Lesson"

@app.route("/healthz")
def health_check():
    end = time.time()
    duration = end - start
    if duration > 45:
        return Response("{'lesson':'k8s'}", status=200)

if __name__== '__main__':
    app.run(host="0.0.0.0", port=80)
```

- To try the HTTP readiness check, create a deployment and service:

```bash
kubectl apply -f http-readiness.yaml
```

- View the deployment, service and endpoint.

```bash
kubectl get deployment
kubectl get pod
kubectl get svc
kubectl get ep
kubectl describe ep readiness-http
```

- Before 45 seconds, view endpoint `NotReadyAddresses` fields to verify that the ports are running but they are excluded from endpoint.

- Delete a pod and see one of pod will be not ready for service. After 45 seconds, it will be included by endpoint again.

```bash
kubectl get pod
kubectl delete pod <pod_name>
kubectl get pod
kubectl describe ep readiness-http
```

- Delete the resources.

```bash
kubectl delete -f http-readiness.yaml
```

Resource:

- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/