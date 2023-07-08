## Part 3 - startupProbe

- The kubelet uses startup probes to know when a container application has started. If such a probe is configured, it disables liveness and readiness checks until it succeeds, making sure those probes don't interfere with the application startup. This can be used to adopt liveness checks on slow starting containers, avoiding them getting killed by the kubelet before they are up and running.

- Sometimes, you have to deal with legacy applications that might require an additional startup time on their first initialization. In such cases, it can be tricky to set up liveness probe parameters without compromising the fast response to deadlocks that motivated such a probe. The trick is to set up a `startup probe` with the same command, HTTP or TCP check, with a `failureThreshold * periodSeconds` long enough to cover the worse case startup time.

- Create a `startup.yaml` and input text below.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: startup
  name: startup-http
spec:
  containers:
  - name: liveness
    image: clarusway/startupprobe
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 3
    startupProbe:
      httpGet:
        path: /healthz
        port: 80
      failureThreshold: 5
      periodSeconds: 15
---
apiVersion: v1
kind: Service   
metadata:
  name: startup-svc
spec:
  type: NodePort  
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30001
  selector:
    test: startup
```

> **failureThreshold:** When a probe fails, Kubernetes will try `failureThreshold` times before giving up. 


- In this image (clarusway/startupprobe), for the `first 60 seconds` the container returns a status of 500. Than the container will return a status of `200`. 

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
    if duration > 60:
        return Response("{'lesson':'k8s'}", status=200)

if __name__== '__main__':
    app.run(host="0.0.0.0", port=80)
```

- To try the startup check, create a Pod:

```bash
kubectl apply -f startup.yaml
kubectl get po
kubectl describe pod startup-http
```

- Thanks to the startup probe, the application will have a maximum of 75 seconds (5 * 15 = 75s) to finish its startup. Once the startup probe has succeeded once, the liveness probe takes over to provide a fast response to container deadlocks. If the startup probe never succeeds, the container is killed after 75s and subject to the pod's restartPolicy.

- Delete the pod.

```bash
kubectl delete -f startup.yaml
```
