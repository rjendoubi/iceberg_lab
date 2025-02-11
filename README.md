# Iceberg and SQL Stream Builder Lab

## Summary
This workshop will take you through the new capabilities that have been added to CDP Public Cloud Lakehouse and into the various features of the SqL stream builder.

In this workshop you will learn how to take advantage of Iceberg to support Data Lakehouse initiatives.

**Value Propositions**: Take advantage of Iceberg - CDP’s Open Data Lakehouse, to experience:  
- Better performance  
- Lower maintenance responsibilities  
- Multi-function analytics without having many copies of data  
- Greater control  

It will also 

*Note to admins: Refer to the Setup file containing the recommendations to setup the lab*


## TABLE OF CONTENT
[1. Introduction to the workshop](#1-introduction-to-the-workshop)  
[2. Table Maintenance in Iceberg](#2-table-maintenance-in-Iceberg)  
[3. Iceberg with NiFi and Sql Stream Builder](#3-introduction-to-iceberg-with-nifi)  
[4. Introduction to Iceberg with Sql Stream Builder](#4-introduction-to-iceberg-with-nifi)  


### 1. Introduction to the workshop  
**Goal of the section:   
Check the dataset made available in a database in a csv format and store it all as Iceberg.**  

#### 1. Data Set

Data set for this workshop is the publicly available Airlines data set, which consists of c.80million row of flight information across the United States.  
Schema for the data set is below:Entity-Relation Diagram of tables we use in todays workshop:

Fact table: flights (86mio rows)
Dimension tables: airlines (1.5k rows), airports (3.3k rows), planes (5k rows) and unique tickets (100k rows).

**Dataset airlines schema**  

![Airlines schema](./images/Iceberg_airlinesschema.png)

**Raw format of the flight set**  

Here displayed in a file explorer:

![Raw Data Set](./images/dataset_folder.png)

#### 2. Access the data set in Cloudera Data Warehouse
In this section, we will check that the airlines data was ingested for you: you should be able to query the master database: `airlines_csv`. 
Each participant will then create their own Iceberg databases out of the shared master database.

Make a note of your username, in a CDP Public Cloud workshop, it should be a account from user01 to user50, 
assigned by your workshop presenter.  

Navigate to Data Warehouse service:
  
![Home_CDW](./images/home_cdw.png)

Then an **Impala** Virtual Warehouse and open the SQL Authoring tool HUE. There are two types of virtual warehouses you can create in CDW, 
here we'll be using the type that leverages **Impala** as an engine:  

![Typesofvirtualwarehouses.png](./images/Typesofvirtualwarehouses.png)  


Execute the following in HUE Impala Editor to test that data has loaded correctly and that you have the appropriate access.  
  
![Hue Editor](./images/Hue_editor.png)


```SQL
SELECT COUNT(*) FROM airlines_csv.flights_csv;  
```

**Note: These queries use variables in Hue**

To set the variable value with your username, fill in the field as below:  
![Setqueryvaribale](./images/Set_variable_hue.png)  


![Flights data](./images/Iceberg_Flightsdata.png)  

#### 3. Generating the Iceberg tables

In this section, we will generate the Iceberg database from the pre-ingested csv tables.   

To run several queries in a row in Hue, make sure you select all the queries:  

![Hue_runqueries.png](./images/Hue_runqueries.png)  


Run the below queries to create your own databases and ingest data from the master database.  
  

```SQL
-- CREATE DATABASES
-- EACH USER RUNS TO CREATE DATABASES
CREATE DATABASE ${user_id}_airlines;
CREATE DATABASE ${user_id}_airlines_maint;


-- CREATE HIVE TABLE FORMAT TO CONVERT TO ICEBERG LATER
drop table if exists ${user_id}_airlines.planes;

CREATE TABLE ${user_id}_airlines.planes (
  tailnum STRING, owner_type STRING, manufacturer STRING, issue_date STRING,
  model STRING, status STRING, aircraft_type STRING,  engine_type STRING, year INT 
) 
TBLPROPERTIES ( 'transactional'='false' )
;

INSERT INTO ${user_id}_airlines.planes
  SELECT * FROM airlines_csv.planes_csv;

-- HIVE TABLE FORMAT TO USE CTAS TO CONVERT TO ICEBERG
drop table if exists ${user_id}_airlines.airports_hive;

CREATE TABLE ${user_id}_airlines.airports_hive
   AS SELECT * FROM airlines_csv.airports_csv;

-- HIVE TABLE FORMAT
drop table if exists ${user_id}_airlines.unique_tickets;

CREATE TABLE ${user_id}_airlines.unique_tickets (
  ticketnumber BIGINT, leg1flightnum BIGINT, leg1uniquecarrier STRING,
  leg1origin STRING,   leg1dest STRING, leg1month BIGINT,
  leg1dayofmonth BIGINT, leg1dayofweek BIGINT, leg1deptime BIGINT,
  leg1arrtime BIGINT, leg2flightnum BIGINT, leg2uniquecarrier STRING,
  leg2origin STRING, leg2dest STRING, leg2month BIGINT, leg2dayofmonth BIGINT,
  leg2dayofweek BIGINT, leg2deptime BIGINT, leg2arrtime BIGINT 
);

INSERT INTO ${user_id}_airlines.unique_tickets
  SELECT * FROM airlines_csv.unique_tickets_csv;

-- CREATE ICEBERG TABLE FORMAT STORED AS PARQUET
drop table if exists ${user_id}_airlines.planes_iceberg;

CREATE TABLE ${user_id}_airlines.planes_iceberg
   STORED AS ICEBERG AS
   SELECT * FROM airlines_csv.planes_csv;

-- CREATE ICEBERG TABLE FORMAT STORED AS PARQUET
drop table if exists ${user_id}_airlines.flights_iceberg;

CREATE TABLE ${user_id}_airlines.flights_iceberg (
 month int, dayofmonth int, 
 dayofweek int, deptime int, crsdeptime int, arrtime int, 
 crsarrtime int, uniquecarrier string, flightnum int, tailnum string, 
 actualelapsedtime int, crselapsedtime int, airtime int, arrdelay int, 
 depdelay int, origin string, dest string, distance int, taxiin int, 
 taxiout int, cancelled int, cancellationcode string, diverted string, 
 carrierdelay int, weatherdelay int, nasdelay int, securitydelay int, 
 lateaircraftdelay int
) 
PARTITIONED BY (year int)
STORED AS ICEBERG 
;
```


### 2. Table Maintenance in Iceberg

Under the maintenance database, let's load the flight table partitioned by year.

```SQL

-- [TABLE MAINTENANCE] CREATE FLIGHTS TABLE IN ICEBERG TABLE FORMAT
drop table if exists ${user_id}_airlines_maint.flights;

CREATE TABLE ${user_id}_airlines_maint.flights (
 month int, dayofmonth int, 
 dayofweek int, deptime int, crsdeptime int, arrtime int, 
 crsarrtime int, uniquecarrier string, flightnum int, tailnum string, 
 actualelapsedtime int, crselapsedtime int, airtime int, arrdelay int, 
 depdelay int, origin string, dest string, distance int, taxiin int, 
 taxiout int, cancelled int, cancellationcode string, diverted string, 
 carrierdelay int, weatherdelay int, nasdelay int, securitydelay int, 
 lateaircraftdelay int
) 
PARTITIONED BY (year int)
STORED AS ICEBERG 
;


```

**Partition evolution**: the insert queries below are designed to demonstrate partition evolution
and snapshot feature for time travel 


```SQL
-- LOAD DATA TO SIMULATE SMALL FILES
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 1;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 2;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 3;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 4;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 5;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 6;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 7;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 8;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 9;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 10;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 11;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1995 AND month = 12;

INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 1;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 2;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 3;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 4;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 5;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 6;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 7;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 8;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 9;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 10;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 11;
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv WHERE year = 1996 AND month = 12;
```

#### 1. Partition evolution
Let's look at the file size. For reference only, and because Iceberg will integrate nicely with all the components of the Cloudera Data Platform 
and with different engines, the task can be performed in PySpark, looking like so:  

**In pyspark**  
  
```SQL
SELECT partition,file_path, file_size_in_bytes
FROM ${user_id}_airlines_maint.flights.files order by partition

```

Here we're sticking to the Cloudera Data Warehouse service for simplicity, so we'll perform the task using Impala.

**In Impala**  
  
```SQL
SHOW FILES in ${user_id}_airlines_maint.flights;
```

Make a note of the average file size which should be around 5MB.
Also note the path and folder structure: a folder is a partition, a file is an ingest as we performed them above.

Now, let's alter the table, adding a partition on the month on top of the year.  

```SQL
ALTER TABLE ${user_id}_airlines_maint.flights SET PARTITION SPEC (year, month);
```  
  
Check the partition fields in the table properties
```SQL

DESCRIBE EXTENDED  ${user_id}_airlines_maint.flights
```

![Partitionkeys](./images/Partitionkeys.png)  


Ingest a month worth of data.  
  
```SQL
INSERT INTO ${user_id}_airlines_maint.flights
 SELECT * FROM airlines_csv.flights_csv
 WHERE year <= 1996 AND month <= 1
```  

Let's have another look:   
  
```SQL
SHOW FILES in ${user_id}_airlines_maint.flights;
```  

Will show the newly ingested data, note the path, folder breakdown is different from before, with the additional partitioning over month taking place.


#### 2. Snapshots

From the INGEST queries earlier, snapshots was created and allow the time travel feature in Iceberg.

```SQL
DESCRIBE HISTORY ${user_id}_airlines_maint.flights;  
```  

Make a note of the timestamps for 2 different snapshots, as well as the snapshot id for one, you can then run:

```SQL
DESCRIBE HISTORY ${user_id}_airlines_maint.flights  BETWEEN '${Timestamp1}' AND '${Timestamp2}';
```
Timestamp format looks like this:  
`2024-04-11 09:48:07.654000000`  
  
  
```SQL

SELECT COUNT(*) FROM ${user_id}_airlines_maint.flights FOR SYSTEM_VERSION AS OF ${snapshotid}
SELECT * FROM ${user_id}_airlines_maint.flights FOR SYSTEM_VERSION AS OF ${snapshotid}

```
Snapshot id format looks like:
`3916175409400584430` **with no quotes**

#### 3. ACID V2

https://blog.min.io/iceberg-acid-transactions/
Let update a row.


```SQL
SELECT * FROM  ${user_id}_airlines_maint.flights LIMIT 1
```

Save the values for year, month, tailnum and deptime to be able to identify that row after update.
Example:  
```SQL

SELECT * FROM ${user_id}_airlines_maint.flights WHERE year = 1996 and MOnth = 2 and tailnum = 'N2ASAA'
and deptime = 730

----Now, Let's run an UPDATE Statement with will likely FAIL
UPDATE ${user_id}_airlines_maint.flights SET uniquecarrier = 'BB' 
WHERE year = 1996 and MOnth = 2 and tailnum = 'N2ASAA'
and deptime = 730
```

As Iceberg table are created as V1 by default, you might get an error message. You will be able to migrate the table from Iceberg V1 to V2 using the below query:
```SQL
ALTER TABLE ${user_id}_airlines_maint.flights SET TBLPROPERTIES('format-version'= '2')
---Reperform the UPDATE

UPDATE ${user_id}_airlines_maint.flights SET uniquecarrier = 'BB' 
WHERE year = 1996 and MOnth = 2 and tailnum = 'N2ASAA'
and deptime = 730

---Check that the update worked:
SELECT * FROM ${user_id}_airlines_maint.flights WHERE year = 1996 and MOnth = 2 and tailnum = 'N2ASAA'
and deptime = 730
```

Copy & paste the SQL below into HUE
```SQL
-- TEST PLANES PROPERTIES
DESCRIBE FORMATTED ${user_id}_airlines.planes;
```  

Pay attention to the following properties: 
- Table Type: `EXTERNAL`
- SerDe Library: `org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe`
- Location: `warehouse/tablespace/external/hive/`

The following labs will take you through various CDP Public Cloud to enable you on what will be available to support Data Lakehouse use cases. 
CDP Public Cloud now includes support for Apache Iceberg in the following services: Impala, Flink, SSB, Spark 3, NiFi, and Replication (BDR). 
This makes Cloudera the only vendor to support Iceberg in a multi-hybrid cloud environment. 
Users can develop an Iceberg application once and deploy anywhere.  

**Handy Iceberg Links**  
[Apache Iceberg Documentation (be careful not everything may be supported yet in CDP)](https://iceberg.apache.org/docs/latest/)  
[Impala Iceberg Cheatsheet](https://docs.google.com/document/d/1cusHyLBA7hS5zLV0vVctymoEbUviJi4aT8SfKyIe_Ao/edit?usp=drive_link)  

### 2. Introduction to Iceberg with NiFi  

In this very short lab we are going to use Nifi to load data into Kafka and Iceberg:  
- First, we will use NiFi to ingest an airport route data set (JSON) and send that data to Kafka and Iceberg.  
- Next we will use NiFi to ingest a countries data set (JSON) and send to Kafka and Iceberg.   
- Finally we will use NiFi to ingest an airports data set (JSON) and send to Kafka and Iceberg.   
  
#### 1. Setup 1 - Create the table in Hue
While still in Hue, please run the below to create Iceberg tables as destination for the Nifi flow will deploy just after:

```SQL
-- TABLES NEEDED FOR THE NIFI LAB
DROP TABLE IF EXISTS ${user_id}_airlines.`routes_nifi_iceberg`;
CREATE TABLE ${user_id}_airlines.`routes_nifi_iceberg` (
  `airline_iata` VARCHAR,
  `airline_icao` VARCHAR,
  `departure_airport_iata` VARCHAR,
  `departure_airport_icao` VARCHAR,
  `arrival_airport_iata` VARCHAR,
  `arrival_airport_icao` VARCHAR,
  `codeshare` BOOLEAN,
  `transfers` BIGINT,
  `planes` ARRAY<VARCHAR>
) STORED AS ICEBERG;

DROP TABLE IF EXISTS ${user_id}_airlines.`airports_nifi_iceberg`;
CREATE TABLE ${user_id}_airlines.`airports_nifi_iceberg` (
  `city_code` VARCHAR,
  `country_code` VARCHAR,
  `name_translations` STRUCT<`en`:string>,
  `time_zone` VARCHAR,
  `flightable` BOOLEAN,
  `coordinates` struct<`lat`:DOUBLE, `lon`:DOUBLE>,
  `name` VARCHAR,
  `code` VARCHAR,
  `iata_type` VARCHAR
) STORED AS ICEBERG;

DROP TABLE IF EXISTS ${user_id}_airlines.`countries_nifi_iceberg`;
CREATE TABLE ${user_id}_airlines.`countries_nifi_iceberg` (
  `name_translations` STRUCT<`en`:VARCHAR>,
  `cases` STRUCT<`su`:VARCHAR>,
  `code` VARCHAR,
  `name` VARCHAR,
  `currency` VARCHAR
) STORED AS ICEBERG;
```


#### 2. Setup 2 - Collect all the configuration details for the flow

You'll need a few information from the workspace to configure the pre-designed flow:
- Your keytab
- Kafka Endpoints
- Hive Metastore URI


**Download you Kerberos Keytab**  
On the left hand menu, click on your username and access the Profile menu. On the right, under Actions, click Get keytab. 
Access your user profile
![Userprofile.png](./images/Userprofile.png)  

Download the keytab file
![Get Keytab](./images/Iceberg_GetKeytab.png)  


**Collect the Kafka Broker endpoints**
In CDP Public Cloud, Kafka is deployed in a Datahub, which is a step previously setup by the lab admin.
![Datahubs](./images/AccessDataHub.png)  
The Endpoints are available on the overview page of the Datahub indicated by the admin, on the bottom menu,
under "endpoints".  

Kafka Endpoints in Datahub overview
![Kafka Borker Endpoints](./images/Iceberg_KafkaBorkerEndpoints.png)


**Grab the Hive Metastore URI**
The hive metastore for the datalake is indicated in the configuration file:  
  
Access the Management Console:  

![AccessManagementConsole.png](./images/AccessManagementConsole.png)  


Select the Datalake:  

![Datalake.png](./images/Datalake.png)  


Access the url for the Cloudera Manager of the environment:  

![ClouderaManagerinfo.png](./images/ClouderaManagerinfo.png)  


Access the Hive metastore service in Cloudera Manager: 

![AccessHiveMetastoreservice.png](./images/AccessHiveMetastoreservice.png)  

Download the Configuration files in a zip:  

![DowloadHiveconfigurationzip.png](./images/DowloadHiveconfigurationzip.png)

In the hive-conf.xml file, grab the value for the hive.metastore.uris
![hive-conf.png](./images/hive-conf.png)  

Hive Metastore URI example:

`thrift://workshopforbt-aw-dl-master0.workshop.vayb-xokg.cloudera.site:9083`


#### 3. Deploy the Nifi Flow

Access the Cloudera Data Flow Service:  
![AccessCDF](./images/AccessCDF.png)  
Let's deploy our NiFi flows. Access the data catalog and identify the `SSB Demo - Iceberg` Flow 

![CataloginCDF.png](./images/CataloginCDF.png)  

and deploy it:  

![CDFDeployFlow.png](./images/CDFDeployFlow.png)

The deployment can take place in on of several CDP environment coexisting in the same CDP tenant, here called **Target Workspace** for this lab, we only have the one 
but a typical usecase would be a dev a test or a production environment.

In the Flow deployment wizard, pick a name for your flow (indicate your usename to avoid confusion with other participant's flow).
You can pick a project at this stage or leave it "Unassigned". Projects are used to limit the visibility of drafts, deployments, Inbound Connections, and custom NAR Configurations within an Environment.
We'll be using the default version of Nifi.
  
  
In the parameter's page, fill in the fields with the value for the parameters necessary to configure the flow.

|Parameter|Value|
|----------|----------|
|CDP Workload User|<enter your user id>|
|CDP Workload User Password|<enter your login password> and set sensitive to ‘Yes’|
|Hadoop Configuration Resources|/etc/hive/conf/hive-site.xml,/etc/hadoop/conf/core-site.xml,/etc/hive/conf/hdfs-site.xml|
|Hive Metastore URI|Collected earlier. Ex:`thrift://workshopforbt-aw-dl-master0.workshop.vayb-xokg.cloudera.site:9083`|
|Kafka Broker Endpoint|Collected earlier. Ex `bt-kafka-corebroker2.workshop.vayb-xokg.cloudera.site:9093, bt-kafka-corebroker1.workshop.vayb-xokg.cloudera.site:9093, bt-kafka-corebroker0.workshop.vayb-xokg.cloudera.site:9093`|  
|Kerberos Keytab|Load file collected earlier|  
  
Leave the default settings for Sizing and scaling, as well as the KPIs and deploy your flow.
  
Give the flow around 10 minutes to deploy.
Once done, you can access the flow within the Nifi Canvas by seletion "View in Nifi":  

![CDFViewInNifi.png](./images/CDFViewInNifi.png)
  
In Hue, you can query the tables previously created and see the data being ingested from Nifi.
```SQL
SELECT * FROM ${user_id}_airlines.`countries_nifi_iceberg`
SELECT * FROM ${user_id}_airlines.`airports_nifi_iceberg`
SELECT * FROM ${user_id}_airlines.`routes_nifi_iceberg`
```
In Kafka, accessing the "Streams Messaging Light Duty" Datahub, powered by Kafka, you can see the topics created by the client in the Nifi processor and the messages ingested subsequently.

![AccessStreamMessengingManager.png](./images/AccessStreamMessengingManager.png)

### 4. Introduction to Iceberg with Sql Stream Builder  
Once we are complete with NiFi, we will shift into Sql Streams Builder to show its capability to query Kafka with SQL,
Infer Schema, Create Iceberg Connectors,  and use SQL to INSERT INTO an Iceberg Table.  
Finally we will wrap up by jumping back into Hue and taking a look at the tables we created.


Access the SSB datahub indicated by the workshop presenter and perform the below steps:

1. Import this repo as a project in Sql Stream Builder
2. Open your Project and have a look around at the left menu. Notice all hover tags. Explore the vast detail in Explorer menus.
3. Import Your Keytab
4. Check out Source Control. If you created vs import on first screen you may have to press import here still. You can setup Authentication here.
5. Create and activate an Environment with a key value pair for your userid -> username
6. Inspect/Add Data Sources. You may need to re-add Kafka. The Hive data source should work out of the box.
7. Inspect/Add Virtual Kafka Tables. You can edit the existing tables against your kafka data source and correct topics. Just be sure to choose right topics and detect schema before save.


Open the SSB UI by clicking on Streaming SQL Console.
![AccessSSB.png](./images/AccessSSB.png)
  

You'll need:
- A project downloaded from github, pointing to a specific and unique repository to import all the confitguration details
- An active environment in your SSB project to store your userid variable
- The Kafka endpoints you'll be querying
  

#### 1.Setup SSB: Project
Before you can use Streaming SQL Console, you need to create a project where you can submit your SQL jobs and
manage your project resources. Created or imported projects can be shared with other users in Streaming SQL Console. You can invite members
using their Streaming SQL Console username and set the access level to member or administrator.
Projects aim to provide a Software Development Lifecycle (SDLC) for streaming applications in SQL Stream Builder
(SSB): they allow developers to think about a task they want to solve using SSB, and collect all related resources,
such as job and table definitions or data sources in a central place.
  
A project is a collection of resources, static definitions of data sources, jobs with materialized views, virtual tables,
user-defined functions (UDF), and materialized view API keys. These resources are called internal to a project and
can be safely used by any job within the project.

A project can be set up by importing a repository from a github source, which we will do here. Within the Git repository, "Project" would be pointing to a folder
within the github repository containing the files to set up data sources, api keys and jobs within SSB. As this folder name needs to be unique, the hack for this workshop
is that all attendees are pointing to the same git repository but pointing to pre-created folders within it named after their username.  
  
Step 1: click import with the SSB Home UI:  
![SSBProjectImport.png](./images/SSBProjectImport.png)  
  
Step 2: indicate the url to the git repo: `https://github.com/marionlamoureux/iceberg_lab` and indicate the branch`main`.  
    
Step 3: point to the pre-created folder named after your username following the naming convention `SSB-Iceberg-Demo-user<xx>`.  
    
![ALLfoldersforSSB.png](./images/ALLfoldersforSSB.png)
  
Click `Import`  
  
![SSBImportwindow.png](./images/SSBImportwindow.png)  
  

  
#### 2.Setup SSB: activate environment
We'll need a variable containing your username to be pointing to the correct Kafka topics and Iceberg tables named after that username in previous labs.
To set a envrionment variable in your SSB project, you'll need an **active** environment.
Creating an environment file for a project means that users can create a template with variables that could be used to
store environment-specific configuration.
For example, you might have a development, staging and production environment, each containing different clusters,
databases, service URLs and authentication methods. Projects and environments allow you to write the logic and create the resources once, and use template placeholders for values that need to be replaced with the environment
specific parameters.
To each project, you can create multiple environments, but only one can be active at a time for a project.
Environments can be exported to files, and can be imported again to be used for another project, or on another cluster.
While environments are applied to a given project, they are not part of the project. They are not synchronized to Git
when exporting or importing the project. This separation is what allows the storing of environment-specific values, or
configurations that you do not want to expose to the Git repository.
  
![SSBNewenvironment.png](./images/SSBNewenvironment.png)

  
The variable key the project is expecting is:
Key: `userid`
Value: <your username>

![EnvrionmentSaveSSB.png](./images/EnvrionmentSaveSSB.png)
  
#### 2.Setup SSB: Create Kafka Data Store
Create Kafka Data Store by selecting Data Sources in the left pane,
clicking on the three-dotted icon next to Kafka, then selecting New Kafka Data Source.
![KafkaAddsource.png](./images/KafkaAddsource.png)  
  
**Name**: {user-id}_cdp_kafka. 
**Brokers**: (Comma-separated List as shown below)
  
Example: `bt-kafka-corebroker2.workshop.vayb-xokg.cloudera.site:9093, 
  bt-kafka-corebroker1.workshop.vayb-xokg.cloudera.site:9093,
  bt-kafka-corebroker0.workshop.vayb-xokg.cloudera.site:9093`
**Protocol**: SASL/SSL
**SASL Mechanism**: PLAIN.
**SASL Username**: <CDP Username>. 
**SASL Password**: Workload User password set by your admin as defined earlier/

Click on Validate to test the connections. Once successful click on Create.
Create Kafka Table: Create Kafka Table, by selecting Virtual Tables in the left pane by clicking on the three-dotted icon next to it.
Then click on New Kafka Table.  
  
![CreateKafkaTable.png](./images/CreateKafkaTable.png)
  
Configure the Kafka Table using the details below.
Table Name: {user-id}_syslog_data.
Kafka Cluster: <select the Kafka data source you created previously>.
Data Format: JSON.
Topic Name: <select the topic created in Schema Registry>.
![KafkaTableConfig.png](./images/KafkaTableConfig.png)  
    
When you select Data Format as AVRO, you must provide the correct Schema Definition when creating the table for SSB to be able to successfully process the topic data. For JSON tables, though, SSB can look at the data flowing through the topic and try to infer the schema automatically, which is quite handy at times. Obviously, there must be data in the topic already for this feature to work correctly.

Note: SSB tries its best to infer the schema correctly, but this is not always possible and sometimes data types are inferred incorrectly. You should always review the inferred schemas to check if it’s correctly inferred and make the necessary adjustments.

Since you are reading data from a JSON topic, go ahead and click on Detect Schema to get the schema inferred. You should see the schema be updated in the Schema Definition tab.
  

**This step is performed automatically when deploying the project from github:Copy/paste the thrift Hive URI**   
In the Cloudera Data Warehousing service, identify the Hive Virtual Warehouse and copy the JDBC url and keep only the node name in the string:  
`hs2-asdf.dw-go01-demo-aws.ylcu-atmi.cloudera.site`

![JDBCfromHive.png](./images/JDBCfromHive.png)


  
## Modifications to Jobs
Note: current repo should not require any job modifications.

**CSA_1_11_Iceberg_Sample** - Example in CSA 1.11 docs  
No modifications should be required to this job  
**Countries_Kafka** - Select from Kafka Countries, Create IceBerg Table, Insert Results  
Confirm Kafka topic  
**Routes_Kafka** - Select from Kafka Routes, Create IceBerg Table, Insert Results  
Confirm Kafka topic  
**Test_Hue_Tables**  
Confirm source iceberg table exists, check table names, and namespaces.  
**Time_Travel**  
Execute required DESCRIBE in Hue, use SnapShot Ids  


### Top Tips
If you are using different topics w/ different schema, use SSB to get the DDL for topic. Copy paste into the ssb job's create statement and begin modifying to acceptance. Just be careful with complicated schema such as array, struct, etc.
If you are testing CREATE and INSERT in iterations, you should increment all table names per test iteration. Your previous interations will effect next iterations so stay in unique table names.
Use DROP statement with care. It will DROP your Virtual Table, but not necessarily the impala/hive table. DROP those in HUE if needed.
Execution of Jobs:
Warning: These are not full ssb jobs. In these are samples you execute each statements one at a time.

```
Execute the SELECT * FROM kafka_topic. This will confirm that you have results in your kafka topic. Be patient, if this is your first job may take some time (1-2 minutes) to report results.
Execute the CREATE TABLE Statement. This will create the virtual table in ssb_default name space. It will not create the table in IMPALA.
Execute the INSERT INTO SELECT. Be Patient. This will create the impala table and begin reporting results from the kafka topic shortly.
Lastly, execute the final select. These results are from IMPALA.
```

