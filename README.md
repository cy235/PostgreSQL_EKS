# Deploy PostgreSQL on Amazon EKS
In this project, we will deploy PostgreSQL on Amazon EKS using Kubernetes Persistent Volumes (PV) and Statefulset. And test the persistence of data volumes.

## Create PV
We create a PV named `postgresql-pv`, the configuration file is [pg-pv.yaml](https://github.com/cy235/PostgreSQL_EKS/blob/master/pg-pv.yaml).</br>
Execute:
```
kubectl create -f pg-pv.yaml
```

## Create Persistent Volume Claim (PVC)
We create a PVC named `postgresql-pvc`, the configuration file is [pg-pvc.yaml](https://github.com/cy235/PostgreSQL_EKS/blob/master/pg-pvc.yaml).</br>
Execute:
```
kubectl create -f pg-pvc.yaml
```

Then, we can check the created PV and PVC in the following:
```
$ kubectl get pv
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   REASON   AGE
postgresql-pv   10Gi       RWO            Retain           Bound    default/postgresql-pvc   manual                  20s

$ kubectl get pvc
NAME             STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   AGE
postgresql-pvc   Bound    postgresql-pv   10Gi       RWO            manual         13s
```

## Create deployment
Now, we create a deployment, where PVC will connnect the existing PV. The configuration file is [pg-deployment.yaml](https://github.com/cy235/PostgreSQL_EKS/blob/master/pg-deployment.yaml), where the it defines a volume which mount to `/var/lib/postgresql/data` (because the data will be stored in this path when PostgreSQL container executing initdb which initializes the database cluster's default locale and character set encoding).</br>
Then create the deployment in the following:
```
kubectl create -f pg-deployment.yaml
kubectl get deployment -o wide

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES          SELECTOR
postgresql-deployment   1/1     1            1           1m     postgresql   postgres:12.1   app=postgresql
```
Now the pod will also be created
```
$ kubectl get pods

NAME                                     READY   STATUS    RESTARTS   AGE
postgresql-deployment-68b8fc59db-ln5zb   1/1     Running   0          124m
```

## Create Service
In order to access deployment of container, we need to expose the port of the service. Here we set this nodePort as 30432.
We create this service as `postgresql-client-service`, the configuration file is [pg-service.yaml](https://github.com/cy235/PostgreSQL_EKS/blob/master/pg-service.yaml).</br>
We create the service:

```
$ kubectl apply -f pg-service.yaml

$ kubectl get service -o wide

NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP         PORT(S)          AGE    SELECTOR
kubernetes                  ClusterIP      10.100.0.1       <none>              443/TCP          177m   <none>
postgresql-client-service   NodePort       10.100.187.152   <none>               5432:30432/TCP   93m    app=postgresql
```

## Connect Database
Here we use the psql client to connect database in two ways.
Since the external IP is not available now, the first way to connect the database is from the inside of the cluster:
```
$ psql -U postgres -h 10.100.187.152 -p 5432
```
or
```
psql -U postgres -h localhost -p 30432
```
Now we expose the IP of the deployment `postgresql-client-service` as `mypgservice`, which can be visited via a load balancer:
```
$ kubectl get deployments
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
postgresql-deployment   1/1     1            1           40m

$ kubectl expose deployment postgresql-deployment --type=LoadBalancer --name=mypgservice
$ kubectl get service -o wide
NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)          AGE   SELECTOR
kubernetes                  ClusterIP      10.100.0.1       <none>                                                                   443/TCP          96m   <none>
mypgservice                 LoadBalancer   10.100.8.2       aa6335e23793f11ea85bf16ac812bad9-431391209.us-east-1.elb.amazonaws.com   5432:32050/TCP   17s   app=postgresql
postgresql-client-service   NodePort       10.100.187.152   <none>                                                                   5432:30432/TCP   11m   app=postgresql

```
We can connect the database from the outside of the cluster:
```
psql -U postgres -h aa6335e23793f11ea85bf16ac812bad9-431391209.us-east-1.elb.amazonaws.com -p 5432
```
Now, we succefully connect PostgreSQL database

```
Password for user postgres:
psql (10.1, server 12.1 (Debian 12.1-1.pgdg100+1))
WARNING: psql major version 10, server major version 12.
         Some psql features might not work.
Type "help" for help.

postgres=#
```
Next, we will test the persistence pf data volumes. We first create a new database and a table, and input some data, then we delete the pod and see whether the new pod has the previous input data.</br>
First we creat a database named `pgtest` and quit.
```
postgres=# create database pgtest;
postgres=#\q
```
Then reconnect the database `pgtest` from outside of cluster.

```
$ psql -U postgres -h aa6335e23793f11ea85bf16ac812bad9-431391209.us-east-1.elb.amazonaws.com -p 5432 pgtest
```
Create a table named `test` and input some data
```
CREATE TABLE test(
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT    NOT NULL
);
CREATE TABLE
pgtest=# insert into test (id,name) values(1,'an wang');
INSERT 0 1
pgtest=# insert into test (id,name) values(2,'benjamin yi');
INSERT 0 1
pgtest=# insert into test (id,name) values(3,'chen yi');
INSERT 0 1
pgtest=# SELECT * FROM test;
 id |    name     
----+-------------
  1 | an wang
  2 | benjamin yi
  3 | chen yi
(3 rows)
```

## Delete the pod
In the following, we delete the pod created during the deployment
```
kubectl delete pod/postgresql-deployment-68b8fc59db-ln5zb
pod "postgresql-deployment-68b8fc59db-ln5zb" deleted
```

Now, we can observe that a new pg pod is created
```
kubectl get pods

NAME                                     READY   STATUS    RESTARTS   AGE
postgresql-deployment-68b8fc59db-zqqpr   1/1     Running   0          15s
```
Now, we reconnect the database, and see whether the previous table and the input data exist or not 
```
$ ./psql -U postgres -h aa6335e23793f11ea85bf16ac812bad9-431391209 -p 5432 pgtest -c "SELECT * FROM test"

 id |    name     
----+-------------
  1 | an wang
  2 | benjamin yi
  3 | chen yi
(3 rows)
```

As expected, even if the pod is deleted, Kubernetes is able to recover the data via persistent volumes.
