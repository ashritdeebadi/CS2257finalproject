# CS257finalproject

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

By doing this we set up a mongo cluster 


2. Import the Data 

We download the dataset from Kaggle  https://raw.githubusercontent.com/ozlerhakan/mongodb-json-files/master/datasets/city_inspections.json


We create a database 

```
use youtubedb;

db.createUser({
    user: 'ashrit',
    pwd: 'xxxx',
    roles: [{ role: 'readWrite', db:'youtubedb'}]
})

```
We enable sharding and add 'trending_date' as the shard key

```
sh.enableSharding("youtubedb")

sh.shardCollection("youtubedb.youtubestat", { "trending_date" : 1 } )

```

We import the data into the collection 

```
mongoimport -u ashrit -p xxxx --db youtubedb --collection youtubestat --file CAvideos.json --jsonArray --batchSize 100
```

```
mkdir
```


# Setting up AWS API Gateway and AWS Lambda
You will need an account on AWS to follow these steps
AWS REGION: N. Virginia (us-east-1)
## Lambda Configuration:
* Open AWS Console and navigate to AWS Lambda console
* Click on create function. Enter name as "home". 
* Choose runtime as Python 3.6
* Click create function.
* Wait for the function to be created.
* In the Function Code section, Click on the drop down under Code Entry Type and choose upload a .zip file and upload the home.zip file.
* Click save at the top right of the console.
* Repeat the same process for query.zip with the lambda name as "query".

## API Gateway Configuration:
* Open AWS Console and navigate to API Gateway Console.
* Click on Create API
* Click on Build under REST API
* Give a name to the API and click Create API
* Click on Actions > Create Method. Choose 'GET' in the new dropdown that has appeared and click on tickmark.
* Type "home" in Lambda Function and click on save. Click OK for the pop up.
* Click on "Method Response". Open the 200 status code and click "Add Header" and type "Content-Type" and click on tickmark.
* Go back using the "<-Method execution" link at the top.
* Choose Integration Response > Open the response status > Open Header Mappings and edit "Content-Type" header. Type in "'text/html'"(single quotes are necessary) and click on the tickmark.
* Click on Mapping template and add mapping template. Again type 'text/html'. In the textbox on the right, type:
 ```
 "$input.path('$')" (without double quotes)
```
* Click Save
* Click on Actions > Create Resource.
* Enter "query" in resource name. Check the "Enable API Gateway CORS" checkbox and click Create Resource.
* Select the newly created resource and click on Actions > Create Method. Choose 'POST' in the new dropdown that has appeared and click on tickmark.
* Type "query" in Lambda Function and click on save. Click OK for the pop up.
* Click on '/' under resources. Click Actions and choose Enable CORS and click Enable CORS and replace existing CORS headers. Click Yes for the popup.
* Repeat the same procedure for '/query' resource.
* Click on Actions > Deploy API
* In Deployment Stage choose [New Stage]. Enter a name for the stage and click Deploy.
* Click on 'Stages' in the left column and click on the stage that was created under stages.
* Click on Invoke URL to open the webpage. Wait for sometime if the page does not load and try again by clicking the link.

# Results
1. Query 1: Most viewed category
![Query1](/images/query1.png)
2. Query 2: Top 10 videos with highest number of likes grouped by country and date
![Query2](/images/query2.png)
3. Query 3: Top 10 videos with highest number of dislikes grouped by country and date
![Query3](/images/query3.png)
4. Query 4: Top 10 videos with highest number of comments grouped by country and date
![Query4](/images/query4.png)
5. Query 5: Top 10 videos with highest number of engagement grouped by country and date
![Query5](/images/query5.png)
6. Performance Graph
![Graph](/images/graph.JPG)