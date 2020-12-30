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

Find the PostgreSQL port, `31309` in this case.

```console
$ kubectl get svc -n clair
NAME       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
postgres   NodePort   10.43.242.146   <none>        5432:31309/TCP                  6h25m
```

Replace `NODEIP` with the IP address of the k8s node, and NODEPORT with the
service port (`31309` in our case).

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

Find the Clair health port, `32006` in this case.

```console
$ kubectl get svc -n clair
NAME       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
postgres   NodePort   10.43.242.146   <none>        5432:31309/TCP                  6h47m
clair      NodePort   10.43.69.151    <none>        6060:30494/TCP,6061:32006/TCP   6h45m

$ curl -X GET -I http://NODEIP:NODEPORT/health
HTTP/1.1 200 OK
Server: clair
Date: Wed, 30 Dec 2020 20:36:57 GMT
Content-Length: 0
```
