# CS2257finalproject

## Problem Statement :
We will be looking at the performance of a sharded MongoDB cluster with HIVE. Both the databases will be queried in their respective language and they will be timed for the duration of query execution only.

1. Setting up the Mongo Cluster in the AWS :

We create a Mongo Cluster by using 3 AWS EC2 instances and We shard the databases and set up a dhard replication factor of 3 to takle fault tolerance 

```
Open AWS console -> CLick on Services -> EC2 -> Select Launch Instance -> Select Ubuntu Server 18.04 LTS -> Select t2.micro -> Click on Next: Configure Instance Details -> Under number of Instances select 3 -> Click on Next: Add Storage Select 25GB under Volume -> Click on Review and Launch -> Select : Next : Add Tags -> Click on Next:Configure Security Group -> Add 5 Rules: Custom TCP rule  port 27017, 27018, 27019, 27020 27021   Source : Custom  0.0.0.0/0   ->  Click on Review and Launch -> Click on Launch 

Note : Create a new ssh key or you can use your existing ssh key to access the instances 

```

For Windows , using the putty we access the intances 

```
Open Putty -> under  Host name enter    ubuntu@#instance_public_ip  e.g., ubuntu@13.57.181.141 , then click on SSH -> Auth -> select the ssh key  and click on open button 

```

Using https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/ We install mongodb on the instance. Then , 


Create a data directory 
```
sudo mkdir -p /db/config/data

```

Give the directory root access

```
sudo chmod 777 /db/config/data
```

By default mongo takes port 27019 for config server members, even if we dont specify it

We replicate config server  by running the command on their respective instance 

```

sudo mongod --port 27019 --configsvr --replSet configReplSet --dbpath /db/config/data --bind_ip localhost,ec2-3-87-191-219.compute-1.amazonaws.com

sudo mongod --port 27019 --configsvr --replSet configReplSet --dbpath /db/config/data --bind_ip localhost,ec2-18-206-114-250.compute-1.amazonaws.com

sudo mongod --port 27019 --configsvr --replSet configReplSet --dbpath /db/config/data --bind_ip localhost,ec2-3-83-83-153.compute-1.amazonaws.com

```

We sign into any one instance to configure the shard server set by running the following command

```
mongo --host ec2-3-87-191-219.compute-1.amazonaws.com --port 27019

rs.initiate( {
   _id : "configReplSet",
   configsvr: true,
   members: [
      { _id: 0, host: "ec2-3-87-191-219.compute-1.amazonaws.com:27019" },
      { _id: 1, host: "ec2-18-206-114-250.compute-1.amazonaws.com:27019" },
      { _id: 2, host: "ec2-3-83-83-153.compute-1.amazonaws.com:27019" }
   ]
})

```


Now we need to replicate the shards, 

```

sudo mkdir -p /db/shard0/data

sudo chmod 777 /db/shard0/data

mongod --port 27018 --shardsvr --replSet shardReplSet1  --dbpath /db/shard0/data --bind_ip localhost,ec2-3-87-191-219.compute-1.amazonaws.com

mongod --port 27018 --shardsvr --replSet shardReplSet1  --dbpath /db/shard0/data --bind_ip localhost,ec2-18-206-114-250.compute-1.amazonaws.com

mongod --port 27018 --shardsvr --replSet shardReplSet1  --dbpath /db/shard0/data --bind_ip localhost,ec2-3-83-83-153.compute-1.amazonaws.com

```

Now we configure the shard replica set by logging into any one instance 

```
mongo --host ec2-3-87-191-219.compute-1.amazonaws.com --port 27018

rs.initiate(
  {
    _id : "shardReplSet1",
    members: [
      { _id : 0, host : "ec2-3-87-191-219.compute-1.amazonaws.com:27018" },
      { _id : 1, host : "ec2-18-206-114-250.compute-1.amazonaws.com:27018" },
      { _id : 2, host : "ec2-3-83-83-153.compute-1.amazonaws.com:27018" }
    ]
  }
)

```

Bind all the instance 

```
mongos --configdb configReplSet/ec2-3-87-191-219.compute-1.amazonaws.com:27019,ec2-18-206-114-250.compute-1.amazonaws.com:27019,ec2-3-83-83-153.compute-1.amazonaws.com:27019 --bind_ip localhost,ec2-3-87-191-219.compute-1.amazonaws.com
mongo --host ec2-3-87-191-219.compute-1.amazonaws.com --port 27017

sh.addShard( "shardReplSet1/ec2-3-87-191-219.compute-1.amazonaws.com:27018,ec2-18-206-114-250.compute-1.amazonaws.com:27018,ec2-3-83-83-153.compute-1.amazonaws.com:27018")

```

Repeat the above process on other two instances 

```
mongo --host ec2-18-206-114-250.compute-1.amazonaws.com --port 27020

rs.initiate(
  {
    _id : "shardReplSet2",
    members: [
      { _id : 0, host : "ec2-3-87-191-219.compute-1.amazonaws.com:27020" },
      { _id : 1, host : "ec2-18-206-114-250.compute-1.amazonaws.com:27020" },
      { _id : 2, host : "ec2-3-83-83-153.compute-1.amazonaws.com:27020" }
    ]
  }
)


sh.addShard("shardReplSet2/ec2-3-87-191-219.compute-1.amazonaws.com:27020,ec2-18-206-114-250.compute-1.amazonaws.com:27020,ec2-3-83-83-153.compute-1.amazonaws.com:27020")

```


```
mongo --host ec2-3-83-83-153.compute-1.amazonaws.com --port 27021

rs.initiate(
  {
    _id : "shardReplSet3",
    members: [
      { _id : 0, host : "ec2-3-87-191-219.compute-1.amazonaws.com:27021" },
      { _id : 1, host : "ec2-18-206-114-250.compute-1.amazonaws.com:27021" },
      { _id : 2, host : "ec2-3-83-83-153.compute-1.amazonaws.com:27021" }
    ]
  }
)


sh.addShard( "shardReplSet3/ec2-3-87-191-219.compute-1.amazonaws.com:27021,ec2-18-206-114-250.compute-1.amazonaws.com:27021,ec2-3-83-83-153.compute-1.amazonaws.com:27021")

```

```
mkdir
```
