# Clair in Kubernetes

## Create a Kubernetes namespace

```bash
kubectl create namespace clair
```

## Stand up a PostgreSQL server

```bash
cd postgres
kubectl apply -f postgres-configmap.yaml
kubectl apply -f postgres-secret.yaml
kubectl apply -f postgres-storage.yaml
kubectl apply -f postgres-statefulset.yaml
kubectl apply -f postgres-service.yaml
```

## Create a database for Clair using PSQL on a Docker container

Find the PostgreSQL port, `30856` in this case.

```console
$ kubectl get svc -n clair
NAME       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
postgres   NodePort   10.43.57.218   <none>        5432:30856/TCP   23s
```

Replace `NODEIP` with the IP address of the k8s node, and NODEPORT with the
service port (`30856` in our case).

```console
$ docker run -it --rm postgres psql -h NODEIP -p NODEPORT -U postgresadmin postgres
Password for user postgresadmin: admin123
psql (13.1 (Debian 13.1-1.pgdg100+1))
Type "help" for help.

postgres=# create user clairdb with password 'clair123';
CREATE ROLE
postgres=# create database clairdb with owner clairdb;
CREATE DATABASE
postgres=# quit
```

## Stand up the Clair server

```bash
cd ../clair
kubectl create secret generic -n clair clairsecret --from-file=./config.yaml
kubectl apply -f clair-deployment.yaml
kubectl apply -f clair-service.yaml
```

The Clair server will start to populate the database, which reportedly takes
20-30 minutes. During that time the server is available, but the reported
vulnerabilities will be incomplete.

## Test the Clair server

Use the external IP of the `clair` service with port `6061`.

```console
$ kubectl get svc -n clair
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                         AGE
postgres   NodePort       10.43.57.218    <none>         5432:30856/TCP                  2m55s
clair      LoadBalancer   10.43.251.106   192.168.1.30   6060:32441/TCP,6061:30561/TCP   14s

$ curl -X GET -I http://192.168.1.30:6061/health
HTTP/1.1 200 OK
Server: clair
Date: Wed, 30 Dec 2020 23:03:28 GMT
Content-Length: 0
```
