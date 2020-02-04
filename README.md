## Minikube

__step-0__

```bash
# modify start.sh to your taste
./start.sh
```

The MongoDB Enterprise Operator for Kubernetes allows devOps teams to:

* Deploy and run MongoDB Ops Manager on K8s  
* Deploy MongoDB clusters on K8s  

### Initial setup

__step-1__

* Create a namespace for all MongoDB assets.

```bash
kube create namespace mongodb
```

__step-2__

Create Custom Resource Definitions

* MongoDB
* MongoDBUser
* MongoDBOpsManager

```bash
# Download Custom Resource Definitions
curl -O https://raw.githubusercontent.com/mongodb/mongodb-enterprise-kubernetes/master/crds.yaml
kube apply -f crds.yaml
# Download MongoDB Enterprise Operator
curl -O https://raw.githubusercontent.com/mongodb/mongodb-enterprise-kubernetes/master/mongodb-enterprise.yaml
kube apply -f mongodb-enterprise.yaml
```

### Deploy MongoDB Kubernetes Operator

__step-3__

```bash
kube create secret generic ops-manager-admin-secret \
--from-literal=Username="admin@opsmanager.com" \
--from-literal=Password="Passw0rd." \
--from-literal=FirstName="Ops" \
--from-literal=LastName="Manager" \
-n mongodb
```

__step-4__

Deploy MongoDB Ops Manager in a Pod as well as a 3 member MongoDB ReplicaSet for the Ops Manager application database.  Each startup time we vary based on Hardware and quota given to Minikube, however expect to wait 5-10 mins for everything to reach Running status.

If you're on a macbook you'll know its working because the fans should be spinning.

```bash
kube apply -f mongodb-ops-manager-1.yml
# wait a few mins for the objects to create
kube -n mongodb get om -w
# should have these objects
kube -n mongodb get pods -o wide  
NAME                        READY  STATUS             RESTARTS  AGE
mongodb-enterprise-operator 1/1    Running            0         62m
ops-manager-0               1/1    Running            0         7m3s
ops-manager-db-0            1/1    Running            0         9m20s
ops-manager-db-1            1/1    Running            0         8m24s
ops-manager-db-2            1/1    Running            0         7m48s
```

[Ops Manager Resource Docs](https://docs.mongodb.com/kubernetes-operator/v1.4/reference/k8s-operator-om-specification/)
[MongoDB Enterprise Kubernetes Operator](https://github.com/mongodb/mongodb-enterprise-kubernetes)

__step-5__

Open MongoDB Ops Manager and login with the `ops-manager-admin-secret` creds above.  To get the right endpoint for Ops Manager retrieve the node's INTERNAL-IP and NodePort.

```bash
# remember INTERNAL-IP
kube -n mongodb get node -o wide
# remember high-side NODE-PORT, should be something like 3xxxx
kube -n mongodb get service ops-manager-svc-ext
```

Open a Browser to http://INTERNAL-IP:NODE-PORT

![Ops Manager Login](/assets/OpsManagerLogin.png)

__step-6__

Remove the `ops-manager-admin-secret` secret from Kubernetes because you remember it right?

```bash
kube delete secret ops-manager-admin-secret -n mongodb
```

__step-7__

Walk through the Ops Manager setup, accepting the defaults.  Once complete you'll have an Ops Manager almost ready to deal.

![Ops Manager Login](/assets/OpsManagerOverview.png)

### Create

### Ops Manager User

__step-8__

```txt
# User > Account > Public API Access > Generate
-------------------------------------------------
Description: om-main-user-credentials
API Key:     0bbb4b24-bf9e-4685-bd41-f41f59685165
```

```bash
kubectl create secret generic om-main-user-credentials \
  --from-literal="user=admin@opsmanager.com" \
  --from-literal="publicApiKey=0bbb4b24-bf9e-4685-bd41-f41f59685165" \
  -n mongodb
```

__step-9__

```bash
kubectl create configmap ops-manager-connection \
  --from-literal="baseUrl=http://ops-manager-svc.mongodb.svc.cluster.local:8080" \
  --from-literal="projectName=Project0" \
  -n mongodb
```

### Deploy and Use MongoDB Standalone on Kubernetes

__step-10__

```bash
kubectl apply -f mongodb-m0-standalone.yaml
# or kube apply -f mongodb-replicaset.yml
# or kube apply -f mongodb-shared.yml
# wait for the 3 node replicaset to come up...
kubectl -n mongodb get mdb  -w
```

__step-11__

Connect from your local machine to the standalone MongoDB instance

```bash
# get IP of master Kubernetes node (the only node in Minikube)
minikube ip
# get NodePort of m0-standalone-svc-external (31793 below)
kubectl -n mongodb get services
NAME                         TYPE        CLUSTER-IP     PORT(S)
m0-standalone-svc            ClusterIP   None           27017/TCP
m0-standalone-svc-external   NodePort    10.96.67.235   27017:31793/TCP
ops-manager-db-svc           ClusterIP   None           27017/TCP
ops-manager-svc              ClusterIP   None           8080/TCP
ops-manager-svc-ext          NodePort    10.96.144.186  8080:31360/TCP
```

__step-12__

```bash
# connect from local machine
mongo 172.16.182.132:31793
MongoDB shell version v4.2.2
connecting to: mongodb://172.16.182.132:31793/test
MongoDB server version: 4.2.3
MongoDB Enterprise > use todosdb
MongoDB Enterprise > db.todos.insertOne({title: "deploy standalone MongoDB on K8s", complete: true})
MongoDB Enterprise > db.todos.insertOne({title: "deploy MongoDB replicaset on K8s", complete: false})
MongoDB Enterprise > db.todos.insertOne({title: "deploy MongoDB sharded cluster on K8s", complete: false})
MongoDB Enterprise > db.todos.find({complete: false})
MongoDB Enterprise > exit
```

__step-13__

Dump and then remove the standalone instance.

```bash
# dump the todosdb database to your local machine
mongodump --uri "mongodb://172.16.182.132:31793/todosdb" -c todos
kubectl -n mongodb delete mdb/m0-standalone
```

### Deploy and Use MongoDB ReplicaSet on Kubernetes

__step-14__

```bash
kubectl apply -f mongodb-m1-replicaset.yml
kubectl -n mongodb get mdb/m1-replica-set -w
# wait until the ReplicaSet is Running
NAME             TYPE         STATE         VERSION     AGE
m1-replica-set   ReplicaSet   Reconciling   4.2.3-ent   58s
m1-replica-set   ReplicaSet   Running       4.2.3-ent   66s
# verify 3 pods are running for the ReplicaSet
NAME                                           READY   STATUS    
m1-replica-set-0                               1/1     Running
m1-replica-set-1                               1/1     Running
m1-replica-set-2                               1/1     Running
```

__step-15__

This time we're going to issue mongo client commands from within the Pod network.  You can grab the connection string from Ops Manager by clicking the "..." button on the ReplicaSet and then select "Connect to this Instance"...copy the connection string (see screenshot).

![ReplicaSet Connection String](/assets/ReplicaSetConnectionString.png)

Now exec into a shell on the Primary Pod and add some data.

```bash
# You can find the primary from Ops Manager UI, in most cases it will be -0
kubectl -n mongodb exec -it m1-replica-set-0 sh
# Now in the Pod shell...copy the connection string to connect
$ /var/lib/mongodb-mms-automation/mongodb-linux-x86_64-4.2.3-ent/bin/mongo \
  --host m1-replica-set-0.m1-replica-set-svc.mongodb.svc.cluster.local \
  --port 27017
MongoDB shell version v4.2.3
connecting to: mongodb://m1-replica-set-0...blah blah blah
MongoDB Enterprise m1-replica-set:PRIMARY> use todosdb
MongoDB Enterprise m1-replica-set:PRIMARY> db.todos.insertOne({title: "deploy standalone MongoDB on K8s", complete: true})
MongoDB Enterprise m1-replica-set:PRIMARY> db.todos.insertOne({title: "deploy MongoDB replicaset on K8s", complete: true})
MongoDB Enterprise m1-replica-set:PRIMARY> db.todos.insertOne({title: "deploy MongoDB sharded cluster on K8s", complete: false})
MongoDB Enterprise m1-replica-set:PRIMARY> db.todos.find({complete: false})
MongoDB Enterprise m1-replica-set:PRIMARY> db.todos.find({complete: false}).pretty()
{
	"_id" : ObjectId("5e38ebcf1ac70e1e4ff81efe"),
	"title" : "deploy MongoDB sharded cluster on K8s",
	"complete" : false
}
```

__step-16__  

Now remove the ReplicaSet.

```bash
kubectl -n mongodb delete mdb/m1-replica-set
```

### Teardown

```bash
kube delete pvc mongodb-standalone
kube delete pv mongodb-standalone
kube delete namespace mongodb
kube delete storageclass mongodb-standalone
kube delete crd mongodb.mongodb.com
kube delete crd mongodbusers.mongodb.com
kube delete crd opsmanagers.mongodb.com

kube delete secret mongodb-admin-creds
kube delete secret ops-manager-admin-secret
kube delete secret ops-manager-admin-secret

```













### Nice commands to know

```bash
kube config set-context --current --namespace=mongodb
```

```bash
r=https://api.github.com/repos/machine-drivers/docker-machine-driver-vmware
curl -LO $(curl -s $r/releases/latest | grep -o 'http.*darwin_amd64' | head -n1) \
 && install docker-machine-driver-vmware_darwin_amd64 \
  /usr/local/bin/docker-machine-driver-vmware
```















### References

1. [Ops Manager in K8s](https://www.mongodb.com/blog/post/running-mongodb-ops-manager-in-kubernetes)
