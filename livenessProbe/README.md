## Part 1 - Setting up the Kubernetes Cluster

- Launch a Kubernetes Cluster of Ubuntu 20.04 with two nodes (one master, one worker).

>*Note: If you have a problem with the Kubernetes cluster, you can use this link for the lesson.*
>https://www.katacoda.com/courses/kubernetes/playground

- Check if Kubernetes is running and nodes are ready.

```bash
kubectl cluster-info
kubectl get node
```

## Part 2 - livenessProbe

- The kubelet uses liveness probes to know when to restart a container. For example, liveness probes could catch a deadlock, where an application is running, but unable to make progress. Restarting a container in such a state can help to make the application more available despite bugs.

- Create a `http-liveness.yaml` and input text below.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: clarusway/probes
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 3
---
apiVersion: v1
kind: Service   
metadata:
  name: liveness-svc
spec:
  type: NodePort  
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30001
  selector:
    test: liveness
```

- In the configuration file, you can see that the Pod has a single container. 

- The `periodSeconds` field specifies that the kubelet should perform a `liveness probe every 3 seconds`. 

- The `initialDelaySeconds` field tells the kubelet that it should `wait 3 seconds before performing the first probe`. 

- To perform a probe, the kubelet sends an `HTTP GET request` to the server that is running in the container and listening on port 80. If the handler for the server's `/healthz` path returns a success code, the kubelet considers the container to be alive and healthy. If the handler returns a failure code, the kubelet kills the container and restarts it.

- Any code `greater than or equal to 200 and less than 400` indicates `success`. Any other code indicates `failure`.

- For the `first 30 seconds` that the container is alive, the `/healthz` handler returns a status of `200`. After that, the handler returns a status of `500`.

```py
app = Flask(__name__)

start = time.time()

@app.route("/healthz")
def health_check():
    end = time.time()
    duration = end - start
    if duration < 30:
        return Response("{'lesson':'k8s'}", status=200)

if __name__== '__main__':
    app.run(host="0.0.0.0", port=80)
```

- The kubelet starts performing health checks 3 seconds after the container starts. So the first couple of health checks will succeed. But after 30 seconds, the health checks will fail, and `the kubelet will kill and restart the container`.

- To try the HTTP liveness check, create a Pod:

```bash
kubectl apply -f http-liveness.yaml
```

- After 30 seconds, view Pod events to verify that liveness probes have failed and the container has been restarted:

```bash
kubectl get po
kubectl describe pod liveness-http
```

- Delete the pod.

```bash
kubectl delete -f http-liveness.yaml
```

### Define a livenessProbe that executes command in the container

- Another kind of liveness probe that executes the command in the contianer.

- Create a `liveness-exec.yaml` and input text below.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: clarusway/probes
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

- Create the pod with `liveness-exec.yaml` command.

```bash
kubectl apply -f liveness-exec.yaml
```

- In the configuration file, you can see that the Pod has a single Container. 

- The `periodSeconds` field specifies that the kubelet should perform a liveness probe every 5 seconds. 

- The `initialDelaySeconds` field tells the kubelet that it should wait 5 seconds before performing the first probe. 

- To perform a probe, the kubelet executes the command `cat /tmp/healthy` in the target container. If the command succeeds, it returns `0`, and the kubelet considers the container to be alive and healthy. If the command returns a `non-zero value`, the kubelet kills the container and restarts it.

- When the container starts, it executes this command:

```bash
/bin/sh -c "touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600"
```

- For the first 30 seconds of the container's life, there is a /tmp/healthy file. So during the first 30 seconds, the command cat /tmp/healthy returns a success code. After 30 seconds, cat /tmp/healthy returns a failure code.

- view the Pod events.

```bash
kubectl get po
kubectl describe pod liveness-exec
```

- Delete the pod.

```bash
kubectl delete -f liveness-exec.yaml
```

### Define a TCP liveness probe 

### Define a TCP liveness probe 

- A third type of liveness probe uses a `TCP socket`. With this configuration, the kubelet will attempt to open a socket to your container on the specified port. If it can establish a connection, the container is considered healthy, if it can't it is considered a failure.

- Create a `tcp-liveness.yaml` and input text below.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-tcp
spec:
  containers:
  - name: liveness-tcp
    image: mysql
    ports:
    - containerPort: 3306
    env:
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

- As you can see, configuration for a TCP check is quite similar to an HTTP check. The kubelet will run the first liveness probe 15 seconds after the container starts. This will attempt to connect to the mysql container on port 8080. If the liveness probe fails, the container will be restarted.

- To try the TCP liveness check, create a Pod:

```bash
kubectl apply -f tcp-liveness.yaml
```

- After 15 seconds, view Pod events to verify that liveness probes:

```bash
kubectl describe pod liveness-tcp
```

- Change the tcpSocket port to 3306 and try again.

```bash
kubectl delete -f tcp-liveness.yaml
kubectl apply -f tcp-liveness.yaml
kubectl describe pod liveness-tcp
watch kubectl get po
```