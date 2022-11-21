# CECG Reference Application - Java 

## P2P Interface

The P2P interface is how the generated pipelines interact with the repo.
For the CECG reference this follows the [3 musketeers pattern](https://3musketeers.io/) of using:

* Make
* Docker
* Compose

These all need to be installed.

## Structure

### Service

Service source code, using Java with Sprint Boot.

### Functional

Stubbed Functional Tests using [Cucumber JVM](https://cucumber.io/docs/installation/java/)

### NFT

Load tests using [K6](https://k6.io/).

## Running the application locally

### Application

```
make run-local
```

This application is exposed locally on port 8080 as well as being available to the tests when run with make.
This is as they are in the same docker network.

### Functional Tests

```
make stubbed-functional
```

You should see:

```
io.cecg.reference.Tests.hello world returns ok PASSED
```

### Non-Functional Tests

```
make stubbed-nft
```

You should see:

```
     checks.........................: 100.00% ✓ 6581      ✗ 0
     data_received..................: 829 kB  14 kB/s
     data_sent......................: 546 kB  9.0 kB/s
     http_req_blocked...............: avg=156.98µs min=6.95µs   med=35.04µs  max=23.34ms p(90)=117.7µs  p(95)=356.83µs
     http_req_connecting............: avg=47.77µs  min=0s       med=0s       max=14.68ms p(90)=0s       p(95)=0s
     http_req_duration..............: avg=3.54ms   min=205.7µs  med=2.5ms    max=41.28ms p(90)=7.7ms    p(95)=10.01ms
       { expected_response:true }...: avg=3.54ms   min=205.7µs  med=2.5ms    max=41.28ms p(90)=7.7ms    p(95)=10.01ms
     http_req_failed................: 0.00%   ✓ 0         ✗ 6581
     http_req_receiving.............: avg=501.79µs min=49.87µs  med=275.91µs max=24.07ms p(90)=911.58µs p(95)=1.51ms
     http_req_sending...............: avg=561.45µs min=25.83µs  med=168.25µs max=27.79ms p(90)=1.37ms   p(95)=2.51ms
     http_req_tls_handshaking.......: avg=0s       min=0s       med=0s       max=0s      p(90)=0s       p(95)=0s
     http_req_waiting...............: avg=2.48ms   min=103.75µs med=1.49ms   max=39.93ms p(90)=5.65ms   p(95)=7.85ms
     http_reqs......................: 6581    107.83203/s
     iteration_duration.............: avg=1s       min=1s       med=1s       max=1.04s   p(90)=1.01s    p(95)=1.01s
     iterations.....................: 6581    107.83203/s
     vus............................: 14      min=9       max=200
     vus_max........................: 200     min=200     max=200
```

## Running in minikube

### Prereqs

* A minikube cluster i.e. you've run `minikube start`
  * You need to enable the ingress addon by executing: `minikube addons enable ingress` and then follow the instructions from the output e.g. if on mac run `minikube tunnel`
* Kubectl or use `minikube kubectl`

### Registries

You'll need a registry. Register for a [Docker hub](https://hub.docker.com/) account and create a private registry e.g. 
`chbatey/reference-service`
If using Docker Desktop then [login](https://www.docker.com/blog/using-docker-desktop-and-docker-hub-together-part-1/)

Update the Makefile `registry` variable with your newly created registry.

### Edit `service/application.yaml` with `database` and `podinfo` service endpoint

```
downstream:
  url: http://podinfo.reference-service-showcase.svc.cluster.local
  port: 9898
  readTimeoutMs: 5000
  conenctTimeoutMs: 1000

database:
  datasource:
    #    driverClassName: org.postgresql.ds.PGSimpleDataSource
    jdbcUrl: &db_url jdbc:postgresql://database.reference-service-showcase.svc.cluster.local:5432/postgres?useServerPrepStmts=false&rewriteBatchedStatements=true
    username: &db_user postgres
    password: &db_pass password
    maximumPoolSize: 10
```
### Pushing the image

```
make docker-build
make docker-push
```

### Replace reference-deployment container image in `/service/k8s-manifests/deployment.yml`
```
    spec:
      containers:
        - name: reference-service
          image: nzacharia/reference-service:latest
```
### Replace hostpath value in PV  in `/service/k8s-manifests/postgres-pvc-pov.yml` with your own path

```    
storage: 5Gi # Sets PV Volume
accessModes:
- ReadWriteMany
hostPath:
path: "/Users/n.zacharia/core-reference-application-java/db-data"
```


### Deploying the refernce-service , podinfo ,database , cfg ,pv ,pvc

```
kubectl apply -f service/k8s-manifests/namespace.yml
kubectl apply -f service/k8s-manifests/deployment.yml
kubectl apply -f service/k8s-manifests/postgres-config.yml
kubectl apply -f service/k8s-manifests/postgres-deployment.yml
kubectl apply -f service/k8s-manifests/postgres-pvc-pv.yml
kubectl apply -f service/k8s-manifests/deployment.yml
```

The service should be running:

```
kubectl get pods -n reference-service-showcase
podinfo-585cf965bc-jlrfh             1/1     Running   0              34m
postgres-6894f7f4fc-hrzvn            1/1     Running   1 (125m ago)   126m
reference-service-74bb49b5f5-57vqw   1/1     Running   0              34m
```

Deploy the ingress and service:

```
kubectl apply -f service/k8s-manifests/expose.yml
kubectl apply -f service/k8s-manifests/postgres-service.yml
```

```
kubectl get ingress -n reference-service-showcase
NAME                CLASS   HOSTS   ADDRESS        PORTS   AGE
reference-service   nginx   *       192.168.49.2   80      144m
```
```
kubectl get svc -n reference-service-showcase
NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
database            ClusterIP   10.107.92.181    <none>        5432/TCP   47h
podinfo             ClusterIP   10.111.172.123   <none>        9898/TCP   42h
reference-service   ClusterIP   10.99.125.12     <none>        80/TCP     2d20h
```
If on Linux you can now access the service on the IP address (which is the minikube IP).

```
curl localhost:80/service/hello
Hello World!%
```

If this doesn't work ensure you followed the instructions when enabling the minikube ingress addon.

### Run the functional tests against deployed application

This shows how you can run the same tests locally and on a deployed version.

```
SERVICE_ENDPOINT="http://localhost:80/service" ./gradlew functional:test
```

### Run the non-functional tests against deployed application

```
SERVICE_ENDPOINT="http://localhost:80/service" k6 run ./nft/ramp-up/test.js
```

### Swagger
Swagger is embedded in the server and available at http://localhost:80/service/swagger-ui/