# mongodb-kubernetes

This solution demonstrates the deployment of MongoDB on Kubernetes with automated management of MongoDB's replica set.

## Background
A *replica set* in MongoDB is a group of mongod processes that maintain the same data set. Replica sets provide redundancy and high availability, and are the basis for all production deployments. 

A replica set contains several data bearing nodes and optionally one arbiter node. Of the data bearing nodes only one member is the primary node, and the rest are secondary nodes. The primary node receives all write operations, and the secondaries replicate the operations to their own data sets to ensure the secondaries' data sets are identical to that of the primary's. If the primary becomes unavailable, one of the secondaries will be elected as the new primary.

Please note the *replica set* is **not** Kubernetes' *ReplicaSet*.

## Challenge
While it's easy to deploy MongoDB as a StatefulSet to Kubernetes, additional considerations must be given to manage the replica set automatically. Generally, the steps to create a replica set are as follows:
1. Initialize the replica set on the node that's going to be the primary.
2. Add remaining nodes to the replica set.

These steps can be executed manually among a group of MongoDB nodes. However, the manual execution defies the advantage of running MongoDB on Kubernetes. That is the orchestration of the nodes along with their needed replica set. Assuming Kubernetes decides to scale up the MongoDB cluster, we'd want the new node to be added to the existing replica set automatically. Inversely, when a MongoDB cluster scales down, the node to remove must be removed from the replica set first.

## Solution
The solution presented here takes advantange of the container lifecycle hooks, as well as a sidecar container which is dedicated to managing the replica set. Collectively the solution does the following:
- Initializes MongoDB replica set.
- Adds/removes new MongoDB nodes to replica set.

## How to Run
0. Create Namespace
```
kubectl -f namespace.yaml
```
1. Create PVs. 
   - This step is needed if you are running Kubernetes on-prem. 
   - Not needed if you are on GKE.
   - The PVs are created on an NFS mount so please make sure you have NFS set up.
```
kubectl -f pv.yaml
```

2. Create StatefulSet
```
kubectl -f mongo-statefulset.yaml
```

## Scale Up and Verify Replica Set
```
# Scale up
kubectl scale sts mongo -n logging --replicas=3

# Attach to the primary MongoDB node

# Run Mongo client
root@mongo-0:/# mongo
MongoDB shell version v3.6.2
connecting to: mongodb://127.0.0.1:27017
MongoDB server version: 3.6.2
Server has startup warnings:
2018-03-22T20:04:39.106+0000 I CONTROL  [initandlisten]
2018-03-22T20:04:39.106+0000 I CONTROL  [initandlisten] ** WARNING: Access control is not enabled for the database.
2018-03-22T20:04:39.106+0000 I CONTROL  [initandlisten] **          Read and write access to data and configuration is unrestricted.
2018-03-22T20:04:39.106+0000 I CONTROL  [initandlisten] ** WARNING: You are running this process as the root user, which is not recommended.
2018-03-22T20:04:39.106+0000 I CONTROL  [initandlisten]
2018-03-22T20:04:39.106+0000 I CONTROL  [initandlisten]
2018-03-22T20:04:39.106+0000 I CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/enabled is 'always'.
2018-03-22T20:04:39.106+0000 I CONTROL  [initandlisten] **        We suggest setting it to 'never'
2018-03-22T20:04:39.106+0000 I CONTROL  [initandlisten]
2018-03-22T20:04:39.106+0000 I CONTROL  [initandlisten] ** WARNING: /sys/kernel/mm/transparent_hugepage/defrag is 'always'.
2018-03-22T20:04:39.106+0000 I CONTROL  [initandlisten] **        We suggest setting it to 'never'
2018-03-22T20:04:39.106+0000 I CONTROL  [initandlisten]
rs0:PRIMARY>

# Check replica set membership
rs0:PRIMARY> rs.status().members.length
3
rs0:PRIMARY>
```

## Scale Down and Verify Replica Set
```
# Scale down
kubectl scale sts mongo -n logging --replicas=2

# Check replica set membership
rs0:PRIMARY> rs.status().members.length
2
```

