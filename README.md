# CS2257finalproject

## Problem Statement :
We will be looking at the performance of a sharded MongoDB cluster with HIVE. Both the databases will be queried in their respective language and they will be timed for the duration of query execution only.

1. Setting up the Mongo Cluster in the AWS :

We create a Mongo Cluster by using 3 AWS EC2 instances and We shard the databases and set up a dhard replication factor of 3 to takle fault tolerance 

```
Open AWS console -> CLick on Services -> EC2 -> Select Launch Instance -> Select Ubuntu Server 18.04 LTS -> Select t2.micro -> Click on Next: Configure Instance Details -> Under number of Instances select 3 -> Click on Next: Add Storage Select 25GB under Volume -> Click on Review and Launch -> Select : Next : Add Tags -> Click on Next:Configure Security Group -> Add 5 Rules: Custom TCP rule  port 27017, 27018, 27019, 27020 27021   Source : Custom  0.0.0.0/0   ->  Click on Review and Launch -> Click on Launch 

Note : Create a new ssh key or you can use your existing ssh key to access the instances 

```





```
mkdir
```
