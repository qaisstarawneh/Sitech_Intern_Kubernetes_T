# Sitech_intern_Kubernetes

In this task:

Assume the following:
- You have a kubernetes cluster, with 2 namesapces
1- web
2- data
3- health

- Loadbalancer controller exists and you can create loadbalancers using its service type

- All docker images used have entrypoints assigned and you do not need to pass the command

- all docker images used, set config via env vars (be creative with the env vars names)

-------------
Part 1:
Create a manifest that would deploy the image "app:latest" with 2 replicas
app:latest uses port 80 internally, and will be exposed via a load balancer on port 80
app:latest uses and env variable that needs namesapce that its in as value value
app:latest uses env var for redis host and another one for redis password, check part 2. (hint for redis service url, how can you access services on other namespaces without leaving the cluster?)
health check is http, path /health
include readiness, liveness probes
this will need to run inside "web" namespace

Part 2:
Create a manifest that would deploy the image "redis:latest" with 1 replicas
redis:latest uses port 12345 internaly, the service should not be publicly available, and you can use any port you see fit for its cluster service port.
redis:latest uses 1 env var for the password
redis:latest needs 1 pvc mounted to /data/redis, the storage class in the cluster is "fast-sc", size of pvc is 1 gb, the pvc can be used in multiple containers if needed.
No health check, no readiness and no liveness
namespace "data"

part 3:
Assume the following:
health:latest image only needs env var containing a list of hosts,protocols and ports
the format of the env var  for that is "host 1,protocol 1,port 1@host 2,protocol 2,port 2@......."

Create a manifest that would deploy the image "health:latest" on all nodes in cluster.
each pod will need to check on the the app and redis service
no health, liveness or readiness
health:latest does not open any ports.

# 1st create the 3 files:
  
  1.
  ```
  nano app_deployment.yaml
  ```
  #File app_deployment content 
  ```
  apiVersion: v1
kind: Namespace
metadata:
  name: web
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  namespace: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: app:latest
        env:
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: REDIS_HOST
          value: "redis.data.svc.cluster.local"
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: redis-password
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /health
            port: 80
        livenessProbe:
          httpGet:
            path: /health
            port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app-service
  namespace: web
spec:
  selector:
    app: app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer


  ```
 
 2.
  ```
  nano redis_deployment.yaml
  ```
  #File redis_deployment content
   ```
 apiVersion: v1
kind: Namespace
metadata:
  name: data
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
  namespace: data
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:latest
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: redis-password
        ports:
        - containerPort: 12345
        volumeMounts:
        - name: redis-data
          mountPath: /data/redis
      volumes:
      - name: redis-data
        persistentVolumeClaim:
          claimName: redis-pvc
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: data
spec:
  selector:
    app: redis
  ports:
  - protocol: TCP
    port: 12346
    targetPort: 12345
  type: ClusterIP
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: data
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: fast-sc
---
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
  namespace: data
data:
  redis-password: UWFpc3MuMTg3MTMxMw==

```

3.
  ```
  nano health_checker_deployment.yaml
  ```
   #File health_deployment content
   
```
apiVersion: v1
kind: Namespace
metadata:
  name: health
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: health-checker
  namespace: health
spec:
  selector:
    matchLabels:
      app: health-checker
  template:
    metadata:
      labels:
        app: health-checker
    spec:
      containers:
      - name: health-checker
        image: health:latest
        env:
        - name: CHECK_LIST
          value: "app-service.web.svc.cluster.local,TCP,80@redis.data.svc.cluster.local,TCP,12346"
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      hostNetwork: true
```
  
#2nd Apply the manifests to your Kubernetes cluster using kubectl:
  
  ```
  kubectl apply -f app_deployment.yaml
  kubectl apply -f redis_deployment.yaml
  kubectl apply -f health_checker_deployment.yaml
  ```
#3rth Monitor the status of the deployments, services, and DaemonSets:
  -To check the status of the deployments:

  ```
  kubectl -n web get deployments
  kubectl -n data get deployments
  kubectl -n health get daemonsets
  ```
  -To check the status of the services:
  
  ```
  kubectl -n web get services
  kubectl -n data get services
  ```

regarding the hint: (hint for redis service url, how can you access services on other namespaces without leaving the cluster?):

  In the provided manifests, I utilized the Kubernetes internal DNS to access the Redis service in the "data" namespace from the "web" namespace without        leaving the cluster. The internal DNS follows the format (service-name).(namespace).svc.cluster.local.
  
  For the Redis service, the internal DNS address used is:
  ```
  redis.data.svc.cluster.local
  ```
  This address was set as the value for the REDIS_HOST environment variable in the "app:latest" container, allowing it to communicate with the Redis service across namespaces within the same cluster.






  
  
  
