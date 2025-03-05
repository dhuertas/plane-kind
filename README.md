# Plane Self-Hosted in KinD (Kubernetes in Docker)

Here are a few notes on how to get Plane Community Edition self-hosted running in a KinD (Kubernetes in Docker) environment - no Helm required.

Main dependencies:
* docker - https://docs.docker.com/engine/install/
* kompose - https://kompose.io/
* kind - https://kind.sigs.k8s.io/
* kubectl - https://kubernetes.io/docs/reference/kubectl/

## 1. Downloading Plane Community Edition
Follow the installation steps for installing Plane community edition on Docker (https://developers.plane.so/self-hosting/methods/docker-compose#install-community-edition). 

In short, download the setup.sh for the self-hosted version:
```bash
curl -fsSL -o setup.sh https://raw.githubusercontent.com/makeplane/plane/master/deploy/selfhost/install.sh
chmod +x setup.sh
mkdir plane && cd plane
./setup.sh install
```
The `setup.sh install` command creates the `./plane-app` folder with a `docker-compose.yaml` and `plane.env` files.

Adjust the variables in the `./plane-app/plane.env` file, then run `docker compose config` to generate a new yaml file with the environment variables resolved:

```bash
docker compose --env-file=plane.env config > docker-compose-resolved.yaml
```

Add the required ports to each container, so `kompose` creates the corresponding k8s service. To do so, apply the following patch:

```
patch < docker-compose-resolved.patch
```

Patch content `docker-compose-resolved.patch`:
```diff
--- docker-compose-resolved.yaml
+++ docker-compose-resolved.yaml.new
@@ -19,6 +19,11 @@
     image: makeplane/plane-admin:v0.25.0
     networks:
       default: null
+    ports:
+      - mode: host
+        target: 3000
+        published: "3000"
+        protocol: tcp
   api:
     command:
       - ./bin/docker-entrypoint-api.sh
@@ -75,6 +80,11 @@
         source: logs_api
         target: /code/plane/logs
         volume: {}
+    ports:
+      - mode: host
+        target: 8000
+        published: "8000"
+        protocol: tcp
   beat-worker:
     command:
       - ./bin/docker-entrypoint-beat.sh
@@ -155,6 +165,11 @@
     image: makeplane/plane-live:v0.25.0
     networks:
       default: null
+    ports:
+      - mode: host
+        target: 3000
+        published: "3000"
+        protocol: tcp
   migrator:
     command:
       - ./bin/docker-entrypoint-migrator.sh
@@ -233,6 +248,11 @@
         source: pgdata
         target: /var/lib/postgresql/data
         volume: {}
+    ports:
+      - mode: host
+        target: 5432
+        published: "5432"
+        protocol: tcp
   plane-minio:
     command:
       - server
@@ -254,6 +274,11 @@
         source: uploads
         target: /export
         volume: {}
+    ports:
+      - mode: host
+        target: 9000
+        published: "9000"
+        protocol: tcp
   plane-mq:
     deploy:
       replicas: 1
@@ -274,6 +299,11 @@
         source: rabbitmq_data
         target: /var/lib/rabbitmq
         volume: {}
+    ports:
+      - mode: host
+        target: 5672
+        published: "5672"
+        protocol: tcp
   plane-redis:
     deploy:
       replicas: 1
@@ -287,6 +317,11 @@
         source: redisdata
         target: /data
         volume: {}
+    ports:
+      - mode: host
+        target: 6379
+        published: "6379"
+        protocol: tcp
   proxy:
     depends_on:
       api:
@@ -336,6 +371,11 @@
     image: makeplane/plane-space:v0.25.0
     networks:
       default: null
+    ports:
+      - mode: host
+        target: 3000
+        published: "3000"
+        protocol: tcp
   web:
     command:
       - node
@@ -355,6 +395,11 @@
     image: makeplane/plane-frontend:v0.25.0
     networks:
       default: null
+    ports:
+      - mode: host
+        target: 3000
+        published: "3000"
+        protocol: tcp
   worker:
     command:
       - ./bin/docker-entrypoint-worker.sh
```

## 2. Create Kubernetes yaml files

The following command generates all the k8s files for the deployment based on the `docker-compose-resolved.yaml` file:
```bash
mkdir k8s && cd k8s
kompose -f ../docker-compose-resolved.yaml -v convert
```

The command creates several yaml files. Edit the `proxy-service.yaml` to set the proxy service port type to NodePort so it can be accessed externally:
```diff
--- proxy-service.yaml
+++ proxy-service.yaml.new
@@ -12,5 +12,7 @@
     - name: "80"
       port: 80
       targetPort: 80
+      nodePort: 30080
+  type: NodePort
   selector:
     io.kompose.service: proxy
```

Here the port `30080` is used so that later it can be mapped in the KinD cluster configuration.

## 3. Plane in a self-hosted KinD cluster
Create a kind cluster config file `kind_cluster.yaml`:
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: plane
networking:
  apiServerAddress: "127.0.0.1"
  apiServerPort: 6443
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30080
    hostPort: 8080
    listenAddress: 127.0.0.1
    protocol: TCP
  extraMounts:
  - hostPath: /var/run/docker.sock
    containerPath: /var/run/docker.sock
```

Then run:
```bash
kind create cluster --config plane_cluster.yaml
```

Wait for the k8s cluster to be ready (it's a container). A new docker container with image `kindest/node:v1.31.0` should appear in `docker ps`.

## 4. Apply Kubernetes files

In the `./k8s` folder run `kubectl apply` on all yaml files to deploy to KinD:

```bash
for f in `ls -tr`; do kubectl apply -f $f; done
```

The web access will be available shortly. Once all ports are running, open a web browser on the host and navigate to the url: http://localhost:8080.

Here are a few stdouts.

```bash
docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED        STATUS         PORTS                                                 NAMES
5882464e5c9b   kindest/node:v1.31.0   "/usr/local/bin/entr…"   25 hours ago   Up 2 minutes   127.0.0.1:6443->6443/tcp, 127.0.0.1:8080->30080/tcp   plane-control-plane
```

```bash
kubectl get pods
NAME          READY   STATUS      RESTARTS        AGE
admin         1/1     Running     2 (3m ago)      25h
api           1/1     Running     6 (2m19s ago)   25h
beat-worker   1/1     Running     5 (2m18s ago)   25h
live          1/1     Running     2 (3m ago)      25h
migrator      0/1     Completed   0               25h
plane-db      1/1     Running     2 (3m ago)      25h
plane-minio   1/1     Running     2 (3m ago)      25h
plane-mq      1/1     Running     2 (3m ago)      25h
plane-redis   1/1     Running     2 (3m ago)      25h
proxy         1/1     Running     9 (2m29s ago)   25h
space         1/1     Running     2 (3m ago)      25h
web           1/1     Running     2 (3m ago)      25h
worker        1/1     Running     5 (2m19s ago)   25h
```

```bash
kubectl get services
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
admin         ClusterIP   10.96.68.169    <none>        3000/TCP       25h
api           ClusterIP   10.96.45.185    <none>        8000/TCP       25h
kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP        25h
live          ClusterIP   10.96.210.36    <none>        3000/TCP       25h
plane-db      ClusterIP   10.96.251.187   <none>        5432/TCP       25h
plane-minio   ClusterIP   10.96.164.97    <none>        9000/TCP       25h
plane-mq      ClusterIP   10.96.105.61    <none>        5672/TCP       25h
plane-redis   ClusterIP   10.96.227.135   <none>        6379/TCP       25h
proxy         NodePort    10.96.3.31      <none>        80:30080/TCP   25h
space         ClusterIP   10.96.42.186    <none>        3000/TCP       25h
web           ClusterIP   10.96.50.171    <none>        3000/TCP       25h
```
