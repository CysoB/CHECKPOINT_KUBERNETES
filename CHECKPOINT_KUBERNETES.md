# CHECKPOINT KUBERNETES — Minikube + Node.js App

## 1) Kubernetes setup (Minikube)

Start Minikube and verify the node:

```bash
minikube start
kubectl get nodes
```

Enable **Ingress** + **metrics-server** (required for HPA):

```bash
minikube addons enable ingress
minikube addons enable metrics-server
```

## 2) App + Docker

Create a minimal Node.js example:

```bash
mkdir k8s-app && cd k8s-app
```

### `app.js`

```js
const http = require('http');
const fs = require('fs');

const PORT = process.env.PORT || 3000;

http.createServer((req, res) => {
  fs.appendFileSync('/data/log.txt', 'hit\n');
  res.end("Hello Kubernetes\n");
}).listen(PORT);
```

### `package.json`

```json
{
  "name": "k8s-app",
  "version": "1.0.0",
  "main": "app.js"
}
```

### `Dockerfile`

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "app.js"]
```

### Build + push

Build in Minikube’s Docker daemon:

```bash
eval $(minikube docker-env)
docker build -t k8s-app:1.0 .
```

If you use Docker Hub:

```bash
docker tag k8s-app:1.0 <username>/k8s-app:1.0
docker push <username>/k8s-app:1.0
```

## 3) Deployment

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: k8s-app
  template:
    metadata:
      labels:
        app: k8s-app
    spec:
      containers:
      - name: app
        image: k8s-app:1.0
        ports:
        - containerPort: 3000
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secret
        volumeMounts:
        - mountPath: "/data"
          name: storage
        resources:
          requests:
            cpu: "100m"
          limits:
            cpu: "200m"
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: pvc
```

Apply and verify:

```bash
kubectl apply -f deployment.yaml
kubectl get pods
```

## 4) ConfigMap + Secret

```bash
kubectl create configmap app-config --from-literal=PORT=3000
kubectl create secret generic app-secret --from-literal=API_KEY=12345
```

## 5) Service

Create `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: k8s-service
spec:
  selector:
    app: k8s-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
```

Apply:

```bash
kubectl apply -f service.yaml
```

## 6) Ingress

Create `ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: k8s-ingress
spec:
  rules:
  - host: k8s.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: k8s-service
            port:
              number: 80
```

Apply:

```bash
kubectl apply -f ingress.yaml
```

Add this to `/etc/hosts` (Windows: `C:\Windows\System32\drivers\etc\hosts`):

1) Get the Minikube IP:

```bash
minikube ip
```

2) Add a line:

```text
<IP> k8s.local
```

## 7) Persistent Volume

Create `pv.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

Create `pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply:

```bash
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
```

## 8) Autoscaling (HPA)

Create a CPU-based HPA:

```bash
kubectl autoscale deployment k8s-app --cpu-percent=50 --min=2 --max=5
```

Verify:

```bash
kubectl get hpa
```

Generate traffic (load generator pod):

```bash
kubectl run -i --tty load-generator --image=busybox /bin/sh
```

Then in the pod shell:

```sh
while true; do wget -q -O- http://k8s-service; done
```

## 9) Monitoring / Debug

Logs :

```bash
kubectl logs <pod-name>
```

CPU usage:

```bash
kubectl top pods
```

Details:

```bash
kubectl describe pod <pod-name>
```

Events :

```bash
kubectl get events
```
