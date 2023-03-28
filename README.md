# confluent-databricks-ml
Workshop Repository for Confluent &lt;> Databricks ML workshop 

## Workshop Overview 

Today, we will be simulating a pizza chain that is trying to keep customers updated with the latest pizza wait times. Imagine ordering a pizza online and receiving information about projected wait times based on historical data for that time of day for the specific shop you are ordering for, but not having up to date information about the store you are attempting to order from. Pretty bad customer experience right? Today, we're going to fix that by using a combination of two powerful real-time platforms: Confluent Cloud and Databricks Delta Live tables to populate the latest wait times in our application based on the real-time pizza orders and their subsequent completions and/or cancellations.    


## Pre-requisites 

### Confluent Cloud Pre-requisites 

- Sign up for Confluent Cloud at the [Confluent Cloud Sign Up Page](https://confluent.cloud/signup). You will receive $400 in free credits upon sign up. 
- Download the Confluent Cloud CLI from the [install page](https://docs.confluent.io/confluent-cli/current/install.html)
- Download this repo 

### Databricks Pre-requisites 

## Confluent Cloud Set-up

### Spin up required Resources 

##### Setup Environment

Create an environment in Confluent Cloud for the workshop called "confluent-databricks-ml-workshop-env" following the steps outlined [here](https://docs.confluent.io/cloud/current/access-management/hierarchy/cloud-environments.html#add-an-environment). An environment contains Kafka clusters and deployed components such as Connect, ksqlDB, and Schema Registry. You can define multiple environments for a Organizations for Confluent Cloud, and there is no charge for creating or using additional environments. Different departments or teams can use separate environments to avoid interfering with each other.    

When the environment is created, select "Essentials -> Begin Configuration" and choose the cloud provider and region of your choice.    

##### Setup Kafka Cluster

Create a cluster in Confluent Cloud for the workshop called "confluent-databricks-ml-workshop-cluster" following the steps outlined [here](https://docs.confluent.io/cloud/current/clusters/create-cluster.html#create-clusters). You can create clusters using the Confluent Cloud Console, Confluent CLI, and REST API. For a description of cluster types and resource quotas for clusters, see [Confluent Cloud Features and Limits by Cluster Type and Service Quotas for Confluent Cloud](https://docs.confluent.io/cloud/current/quotas/index.html#service-quotas-for-ccloud). The cluster should have the following attributes: 
- Basic cluster in the cloud provider & region of your choice
- Single AZ setup      

##### Setup ksqlDB Cluster

Once the cluster is deployed, create a ksqlDB cluster (with global access & a size of 1 CSU) called "pizza-wait-times-app" following the steps outlined [here](https://docs.confluent.io/cloud/current/get-started/index.html#step-1-create-a-ksql-cloud-cluster-in-ccloud). ksqlDB is fully hosted on Confluent Cloud and provides a simple, scalable, resilient, and secure event streaming platform.   


Create an API key for Schema Registry:     

Returning to the Confluent Cloud Dashboard, select your environment
On the right-hand side you will see a stream governance menu. Under "Credentials", select "Add Key"
Click "Create key" and store your credentials

##### Setup Kafka topics 

We are going to set up the following topics to bring our use case to life: 
- waittimes_avro 
- pizza_orders_avro 
- pizza_orders_cancelled_avro 
- pizza_orders_completed_avro 

First, we need to create the topics listed above following the documentation [here](https://docs.confluent.io/cloud/current/client-apps/topics/manage.html#create-a-topic). An Apache Kafka® topic is a category or feed that stores messages. Producers send messages and write data to topics, and consumers read messages from topics. We will create each topic above with 1 partition to guarantee ordering across the entire topic.     

### Populate Event Data

##### Produce initial wait times

Imagine we have a stale machine learning model that has created some estimated wait times for our pizza orders. Initially, let's load that wait time into our Kafka Cluster. However, once we are done with the lab we will be able to receive the most up to date wait times and communicate these to our customers so they have a real-time update when they attempt to make a pizza order.     

In Kafka we store schemas for our topics in a Schema Registry. In Confluent Cloud you create Schema Registries at the environment level. You can create and edit schemas in a schema editor and associate them with Kafka topics. We will use the Schema Registry created above to store the schema information for our topics. The schema for our waittimes_avro topic can be found in schemas/waittimes-avro-schema.json. As you can see, the AVRO schema is stored in a json file. We will use this AVRO schema to serialize the messages we write to Confluent Cloud.      

We will initialize the data in our waittimes topic by using the Confluent CLI to produce messages to our topic. We will login to the CLI, select the appropriate cluster and create an API key for the cluster.      

Choose the cluster you've created for this workshop:     
```
confluent login --save
confluent environment list 
confluent environment use <env-id> 
confluent kafka cluster list 
confluent kafka cluster use <cluster-id> 
```

Create an API key for the CLI to use:   
```
confluent api-key create --resource <cluster-id>
confluent api-key use <api-key> --resource <cluster-id>
```

Go to the schema directory: 
```
cd schemas
```

Start a producer to write to our tables topic: 
```
confluent kafka topic produce waittimes_avro --value-format avro --schema waittimes-avro-schema.json
```

The producer will start with some information and then wait for you to enter input.
```
Successfully registered schema with ID 100001
Starting Kafka Producer. ^C or ^D to exit
```

Below are records in JSON format with each line representing a single record (each for the stale waittime data). In this case we are producing records in Avro format, however, first they are passed to the producer in JSON and the producer converts them to Avro based on the waittimes-avro-schema.json schema prior to sending them to Kafka.

Copy each line and paste it into the producer terminal, or copy-paste all of them into the terminal and hit enter.
```
{"store_id":1,"wait_time":15}
{"store_id":2,"wait_time":20}
{"store_id":3,"wait_time":7}
{"store_id":4,"wait_time":23}
{"store_id":5,"wait_time":25}
{"store_id":6,"wait_time":20}
{"store_id":7,"wait_time":13}
{"store_id":8,"wait_time":10}
{"store_id":9,"wait_time":2}
{"store_id":10,"wait_time":6}
```

##### Create an API Key to be used for your Connectors 
- In the Confluent Cloud Dashboard in your cluster UI, select Cluster Overview -> API Keys
- Click the "Add Key" Button and select Global Access and create your key. Download your key and save it as we will use this for the connector creation. 

##### Create a connector to bring in pizza orders data
- In the Confluent Cloud Dashboard in your cluster UI, select Connectors
- Click the "+ Add Connector" button
- Select the "Datagen Source" connector 
- Choose the "pizza_orders_avro" topic & click continue
- For your Kafka Credentials, select "Use an Existing Key" and put in the key and secret created above. 
- For your output record value format select "AVRO"
- For your template, click "Show more options" and select "Pizza orders". 
- For sizing leave the default of 1 task selected & click "continue"
- Name your connector "pizza_orders_avro_datagen" and launch your connector. 

##### Create a connector to bring in pizza orders cancelled & completed data
Repeat the above steps for both pizza_orders_cancelled_avro & pizza_orders_completed_avro with the appropriate topics & templates (all other configurations the same). When you are finished you should have 3 deployed Datagen Connectors and you should see data being populated into your topics. 

### Transform your data in real time using ksqlDB

##### Create ksqlDB Streams using your topics 

A stream is a partitioned, immutable, append-only collection that represents a series of historical facts. For example, the rows of a stream could model a sequence of financial transactions, like "Alice sent $100 to Bob", followed by "Charlie sent $50 to Bob". Once a row is inserted into a stream, it can never change. New rows can be appended at the end of the stream, but existing rows can never be updated or deleted.     

Each row is stored in a particular partition. Every row, implicitly or explicitly, has a key that represents its identity. All rows with the same key reside in the same partition.     

- In the Confluent Cloud Dashboard in your Cluster UI, select ksqlDB and select your application 
- Select the "Streams" Tab in the application 
- Click the "Import topics as streams" and import all the topics you created in the above steps


##### Inspect the data in your topics using Queries 
There are three kinds of queries in ksqlDB: persistent, push, and pull. Each gives you different ways to work with rows of events.       

A push query is a form of query issued by a client that subscribes to a result as it changes in real-time. A good example of a push query is subscribing to a particular user's geographic location. The query requests the map coordinates, and because it's a push query, any change to the location is "pushed" over a long-lived connection to the client as soon as it occurs. This is useful for building programmatically controlled microservices, real-time apps, or any sort of asynchronous control flow.      

Push queries are expressed using a SQL-like language. They can be used to query either streams or tables for a particular key. Also, push queries aren't limited to key look-ups. They support a full set of SQL, including filters, selects, group bys, partition bys, and joins. Push queries will contain the command "EMIT CHANGES" to notify the ksqlDB query to remain continuously running.      

Navigate to the "Editor" tab in your ksqlDB application. Let's write a query to filer out the pizza orders for a single store:       
```
SELECT *
FROM  PIZZA_ORDERS_AVRO 
WHERE STORE_ID=2
EMIT CHANGES;
```

## Databricks Setup

Refer to notebooks sent via email for instructions.    
