# mongodb-replica-set OnPremise (Vagrant)

**Mongo Replica Sets**

Run MongoDB Replica Set on Kubernetes using Statefulset and PersistentVolumeClaim.


**Create Secret Key file**

```
$ openssl rand -base64 750 > key.txt
```

```
$ kubectl create secret generic shared-bootstrap-data --from-file=internal-auth-mongodb-keyfile=key.txt
secret "shared-bootstrap-data" created
```

**Create PersistentVolumes**

Create on worker-01: ```mkdir -p /mnt/mongo-disk-01```
          worker-02: ```mkdir -p /mnt/mongo-disk-02```
          worker-03: ```mkdir -p /mnt/mongo-disk-03```

then run:

```
$ kubect apply -f volume01.yaml
$ kubect apply -f volume02.yaml
$ kubect apply -f volume03.yaml

$ kubectl get pv
NAME                                        STATUS    VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongodb-persistent-storage-claim-mongod-0   Bound     volume01   5Gi        RWO                           5h56m
mongodb-persistent-storage-claim-mongod-1   Bound     volume02   5Gi        RWO                           5h52m
mongodb-persistent-storage-claim-mongod-2   Bound     volume03   5Gi        RWO                           5h50m

``` 


**Deploy MongoDB ReplicaSet YAML file**

```
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  labels:
    name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    role: mongo
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongod
spec:
  serviceName: mongodb-service
  replicas: 3
  selector:
    matchLabels:
      role: mongo
  template:
    metadata:
      labels:
        role: mongo
    spec:
      containers:
      - name: mongod-container
        image: mongo:4.2
        command:
        - "numactl"
        - "--interleave=all"
        - "mongod"
        - "--bind_ip"
        - "0.0.0.0"
        - "--replSet"
        - "MainRepSet"
        - "--auth"
        - "--clusterAuthMode"
        - "keyFile"
        - "--keyFile"
        - "/etc/secrets-volume/internal-auth-mongodb-keyfile"
        - "--setParameter"
        - "authenticationMechanisms=SCRAM-SHA-1"
        resources:
          requests:
            cpu: 0.2
            memory: 200Mi
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: secrets-volume
          readOnly: true
          mountPath: /etc/secrets-volume
        - name: mongodb-persistent-storage-claim
          mountPath: /data/db
      volumes:
      - name: secrets-volume
        secret:
          secretName: shared-bootstrap-data
          defaultMode: 256
  volumeClaimTemplates:
  - metadata:
      name: mongodb-persistent-storage-claim
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 2Gi
```
**Deploy the YAML file**

```
$ kubect apply -f mongodb-rs.yaml
service/mongodb-service created
statefulset.apps/mongod created
```
**Check Running configuration and Pods**

```
$ kubect get all
NAME           READY   STATUS    RESTARTS   AGE
pod/mongod-0   1/1     Running   0          42m
pod/mongod-1   1/1     Running   0          42m
pod/mongod-2   1/1     Running   0          42m

NAME                      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
service/kubernetes        ClusterIP   10.96.0.1    <none>        443/TCP     8h
service/mongodb-service   ClusterIP   None         <none>        27017/TCP   5h3m

NAME                      READY   AGE
statefulset.apps/mongod   3/3     5h3m

```

**Setup ReplicaSet Configuration**

Now we're ready to configure the ReplicaSet, so let's connect to one of our mongodb pod and In the shell initiate the replica set:

```
$ kubectl exec -it mongod-0 bash
$ mongo
> rs.initiate({_id: "MainRepSet", version: 1, members: [
       { _id: 0, host : "mongod-0.mongodb-service.default.svc.cluster.local:27017" },
       { _id: 1, host : "mongod-1.mongodb-service.default.svc.cluster.local:27017" },
       { _id: 2, host : "mongod-2.mongodb-service.default.svc.cluster.local:27017" }
 ]});
 
```
checking the status of the replica set

```
> rs.status();
{
	"set" : "MainRepSet",
	"date" : ISODate("2019-09-23T20:20:11.809Z"),
	"myState" : 1,
	"term" : NumberLong(18),
	"syncingTo" : "",
	"syncSourceHost" : "",
	"syncSourceId" : -1,
	"heartbeatIntervalMillis" : NumberLong(2000),
	"optimes" : {
		"lastCommittedOpTime" : {
			"ts" : Timestamp(1569270005, 1),
			"t" : NumberLong(18)
		},
		"lastCommittedWallTime" : ISODate("2019-09-23T20:20:05.832Z"),
		"readConcernMajorityOpTime" : {
			"ts" : Timestamp(1569270005, 1),
			"t" : NumberLong(18)
		},
		"readConcernMajorityWallTime" : ISODate("2019-09-23T20:20:05.832Z"),
		"appliedOpTime" : {
			"ts" : Timestamp(1569270005, 1),
			"t" : NumberLong(18)
		},
		"durableOpTime" : {
			"ts" : Timestamp(1569270005, 1),
			"t" : NumberLong(18)
		},
		"lastAppliedWallTime" : ISODate("2019-09-23T20:20:05.832Z"),
		"lastDurableWallTime" : ISODate("2019-09-23T20:20:05.832Z")
	},
	"lastStableRecoveryTimestamp" : Timestamp(1569269945, 1),
	"lastStableCheckpointTimestamp" : Timestamp(1569269945, 1),
	"members" : [
		{
			"_id" : 0,
			"name" : "mongod-0.mongodb-service.default.svc.cluster.local:27017",
			"ip" : "192.168.171.6",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 3300,
			"optime" : {
				"ts" : Timestamp(1569270005, 1),
				"t" : NumberLong(18)
			},
			"optimeDate" : ISODate("2019-09-23T20:20:05Z"),
			"syncingTo" : "",
			"syncSourceHost" : "",
			"syncSourceId" : -1,
			"infoMessage" : "",
			"electionTime" : Timestamp(1569266723, 1),
			"electionDate" : ISODate("2019-09-23T19:25:23Z"),
			"configVersion" : 1,
			"self" : true,
			"lastHeartbeatMessage" : ""
		},
		{
			"_id" : 1,
			"name" : "mongod-1.mongodb-service.default.svc.cluster.local:27017",
			"ip" : "192.168.37.197",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 3286,
			"optime" : {
				"ts" : Timestamp(1569270005, 1),
				"t" : NumberLong(18)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1569270005, 1),
				"t" : NumberLong(18)
			},
			"optimeDate" : ISODate("2019-09-23T20:20:05Z"),
			"optimeDurableDate" : ISODate("2019-09-23T20:20:05Z"),
			"lastHeartbeat" : ISODate("2019-09-23T20:20:10.239Z"),
			"lastHeartbeatRecv" : ISODate("2019-09-23T20:20:11.430Z"),
			"pingMs" : NumberLong(0),
			"lastHeartbeatMessage" : "",
			"syncingTo" : "mongod-2.mongodb-service.default.svc.cluster.local:27017",
			"syncSourceHost" : "mongod-2.mongodb-service.default.svc.cluster.local:27017",
			"syncSourceId" : 2,
			"infoMessage" : "",
			"configVersion" : 1
		},
		{
			"_id" : 2,
			"name" : "mongod-2.mongodb-service.default.svc.cluster.local:27017",
			"ip" : "192.168.202.198",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 3299,
			"optime" : {
				"ts" : Timestamp(1569270005, 1),
				"t" : NumberLong(18)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1569270005, 1),
				"t" : NumberLong(18)
			},
			"optimeDate" : ISODate("2019-09-23T20:20:05Z"),
			"optimeDurableDate" : ISODate("2019-09-23T20:20:05Z"),
			"lastHeartbeat" : ISODate("2019-09-23T20:20:10.239Z"),
			"lastHeartbeatRecv" : ISODate("2019-09-23T20:20:11.091Z"),
			"pingMs" : NumberLong(0),
			"lastHeartbeatMessage" : "",
			"syncingTo" : "mongod-0.mongodb-service.default.svc.cluster.local:27017",
			"syncSourceHost" : "mongod-0.mongodb-service.default.svc.cluster.local:27017",
			"syncSourceId" : 0,
			"infoMessage" : "",
			"configVersion" : 1
		}
	],
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1569270005, 1),
		"signature" : {
			"hash" : BinData(0,"wI2K/ct8OemVhmlROAhlPsE4mIA="),
			"keyId" : NumberLong("6739876011908792322")
		}
	},
	"operationTime" : Timestamp(1569270005, 1)
} 
```
As you can see mongod-0 is PRIMARY and the others two mongod-1 and mongod-2 are set to SECONDARY

**Let's create now the Admin user**

From mongodb shell type:

```
> db.getSiblingDB("admin").createUser({
      user : "my_admin",
      pwd  : "myadminpassword123",
      roles: [ { role: "root", db: "admin" } ]
 });
 ```
 
 **Now trying to insert some data**
 
 from mongod-0 PRIMARY shell type:
 
 ```
 > db.getSiblingDB('admin').auth("my_admin", "myadminpassword123");
> use mydb;
> db.mycollection.insert({R1:"YAMAHA"});
> db.mycollection.insert({ZX10R:"KAWASAKI"});
> db.mycollection.insert({PANIGALEV4R:"DUCATI"});
> db.mycollection.find();
{ "_id" : ObjectId("5d892b979c03797a93746efb"), "R1" : "YAMAHA" }
{ "_id" : ObjectId("5d892bad9c03797a93746efc"), "ZX10R" : "KAWASAKI" }
{ "_id" : ObjectId("5d892bbe9c03797a93746efd"), "PANIGALEV4R" : "DUCATI" }

```

**Check Data Integrity from SECONDARY node**

```
$ kubectl exec -ti mongod-1 bash
$ mongo
> db.getSiblingDB('admin').auth("my_admin", "myadminpassword123");
> db.getMongo().setSlaveOk()
> use mydb;
> db.mycollection.find();
{ "_id" : ObjectId("5d892b979c03797a93746efb"), "R1" : "YAMAHA" }
{ "_id" : ObjectId("5d892bad9c03797a93746efc"), "ZX10R" : "KAWASAKI" }
{ "_id" : ObjectId("5d892bbe9c03797a93746efd"), "PANIGALEV4R" : "DUCATI" }

``` 
**Test Persistentvolumes Data**

Delete all stack

```
$ kubect delete -f mongodb-rs.yaml
service "mongodb-service" deleted
statefulset.apps "mongod" deleted
```
Recreate MongoDB ReplicaSet

```
$ kubectl apply -f mongodb-rs.yaml
service/mongodb-service created
statefulset.apps/mongod created
```

Check Integrity data

Connect to one of the mongodb pod. 
```
$ kubectl exec -ti mongodb-0 bash
$ mongo
```

If node is PRIMARY type:

```
> db.getSiblingDB('admin').auth("my_admin", "myadminpassword123");
> use mydb;
> db.mycollection.find();
{ "_id" : ObjectId("5d892b979c03797a93746efb"), "R1" : "YAMAHA" }
{ "_id" : ObjectId("5d892bad9c03797a93746efc"), "ZX10R" : "KAWASAKI" }
{ "_id" : ObjectId("5d892bbe9c03797a93746efd"), "PANIGALEV4R" : "DUCATI" }

```
If node is SECONDARY type:

```
db.getSiblingDB('admin').auth("my_admin", "myadminpassword123");
db.getMongo().setSlaveOk()
use mydb;
db.mycollection.find();
{ "_id" : ObjectId("5d892b979c03797a93746efb"), "R1" : "YAMAHA" }
{ "_id" : ObjectId("5d892bad9c03797a93746efc"), "ZX10R" : "KAWASAKI" }
{ "_id" : ObjectId("5d892bbe9c03797a93746efd"), "PANIGALEV4R" : "DUCATI" }

```

Since we didn't delete the PVC we still have all the data accessible through all nodes.
