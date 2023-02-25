# datastreaming
Apache Flink data streaming using Kafka/RedPanda . In this example we will consume the sample airtraffic data using opensky api and publish it to 'redpanda' cluster thru kafka connect .

Once the data is available in RedPanda cluster , we can stream it using Apache Flink and write Spark SQL to do stream analytics



![image](https://user-images.githubusercontent.com/64332344/210473707-00454559-f378-482a-829a-9fdb54ad345f.png)


## Below are the steps to run the application
1) Install Docker Desktop
2) Create an account with opensky ( https://opensky-network.org/)
3) Checkout and build the Kafka OpenSky connector, command and details at (https://github.com/nbuesing/kafka-connect-opensky) 
4) Download and install kafka (https://kafka.apache.org/downloads) , i used the binary download . After download you will see the below directory

![image](https://user-images.githubusercontent.com/64332344/219564252-cc8d575f-3a66-40d9-b5a3-3d70f465b19c.png)

(This required to run the kafka connect plugin, basically we will install the opensky kafka connect plugin in Kakfa to run it.)

5) Next step is to start the opensky kafka connecter , to do this we need the below config changes

- Update the *plugin.path* to the location of the build output from step #4 above
```
offset.storage.file.filename=/tmp/connect.offsets
# Flush much faster than normal, which is useful for testing/debugging
offset.flush.interval.ms=10000

# Set to a list of filesystem paths separated by commas (,) to enable class loading isolation for plugins
# (connectors, converters, transformations). The list should consist of top level directories that include 
# any combination of: 
# a) directories immediately containing jars with plugins and their dependencies
# b) uber-jars with plugins and their dependencies
# c) directories immediately containing the package directory structure of classes of plugins and their dependencies
# Note: symlinks will be followed to discover dependencies or plugins.
# Examples: 
# plugin.path=/usr/local/share/java,/usr/local/share/kafka/plugins,/opt/connectors,
plugin.path=/Users/xyz/kafka-connect-opensky-master/build/distributions
```
- Update the Redpanda cluster (sink) and Opensky (source) . Make sure you update the username and password for opensky that you created in step #2

```
connector.class=com.github.nbuesing.kafka.connect.opensky.OpenSkySourceConnector

name=opensky
tasks.max=1
topic=flights_json2

key.converter=org.apache.kafka.connect.storage.StringConverter

value.converter=org.apache.kafka.connect.json.JsonConverter
value.converter.schemas.enable=false

#value.converter=io.confluent.connect.avro.AvroConverter
#value.converter.schema.registry.url=http://localhost:8081

interval=900

#bounding.boxes=-90.0 0.0 -180.0 0.0, 0.0 90.0 -180.0 0.0, -90.0 0.0 0.0 180.0, 0.0 90.0 0.0 180.0
bounding.boxes=45.8389 47.8229 5.9962 10.5226 , 24.396308 49.384358 -124.848974 -66.885444
#bounding.boxes=45.8389 47.8229 5.9962 10.5226
#offset.storage.file.filename=/tmp/converter.offsets

#opensky.url=http://localhost:9999/api
opensky.url=https://opensky-network.org/api/

opensky.timeout.connect=30s
opensky.timeout.read=30s
opensky.username=<username>
opensky.password=<password>

transforms=flatten,rename

transforms.flatten.type=org.apache.kafka.connect.transforms.Flatten$Value
transforms.flatten.delimiter=_

transforms.rename.type=org.apache.kafka.connect.transforms.ReplaceField$Value
transforms.rename.renames=location_lat:latitude,location_lon:longitude


```

6) Before installing the RedPanda cluster , we will configure the Apache Flink connector . Unlike Kafka connect we will run Flink inside docker, Since we are going to use Flink SQL we need to build a new image using the base flink image and install SQL connector plugin.

```
docker build <use the docker file in this repo> -t openskyflink
```
7) Now that we have the flink docker images are ready , you can use the docker compose file to **run Apache Flink & Red panda cluster** in docker.

```
Exceute the below command inside the folder where the docker-compose.yml file is present
 " docker-compose up -d "
```
![image](https://user-images.githubusercontent.com/64332344/221382713-699f8ad9-e43a-494d-bce4-cb1f38ada40b.png)

8) At this point we have all the 3 components running as below 
- Red Panda cluster : Running inside docker 
- OpenSky Kafka connecter : Running as a standalone app in the local machine
- Apache Flink SQL : Running inside docker

9) Use the below command to create the topic on the Red Panda Cluster

```
docker exec -it redpanda-1 rpk topic create flights_json2 --brokers=localhost:9092

```
10) **Run the Kafka connect** using the below command , this should be run from the directory where you installed kafka (step #4). 

```
 bin/connect-standalone.sh config/connect-standalone.properties config/opensky-source.properties
```
11)  Install RedPanda Console , to view the opensky events in the **flights_json2** topic loaded by the kafka connector. Download the respective client binaries from https://github.com/redpanda-data/console/releases and run the below command

```
./redpanda-console -config.filepath=./rp-client.yaml

```

![image](https://user-images.githubusercontent.com/64332344/221383009-193a3ce1-3192-4290-822d-0dacfd323389.png)

12) You can access the Flink management cosole at **http://localhost:8081**
13) Once you verify the data exists on the topic , now it is time to execute the SQL on the **streaming data**

```
docker exec -it flink-jobmanager-1 bash - ./bin/sql-client.sh
```
14) On the SQL console , first step is to create a table that matches the schema on the **flights_json2** data

```
-- create table
CREATE TABLE opensky (
    id STRING,
    callsign STRING, 
    originCountry STRING, 
    timePosition BIGINT,
    lastContact BIGINT, 
    latitude DOUBLE, 
    longitude DOUBLE, 
    barometricAltitude DOUBLE, 
    onGround BOOLEAN,
    velocity DOUBLE,     
    heading DOUBLE, 
    verticalRate DOUBLE, 
    geometricAltitude DOUBLE, 
    squawk VARCHAR,
    specialPurpose BOOLEAN,
    positionSource VARCHAR,
    timeltz AS TO_TIMESTAMP_LTZ(timePosition,3) ,
    WATERMARK FOR timeltz AS timeltz - INTERVAL '1' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'flights_json2',
    'scan.startup.mode' = 'earliest-offset',
    'properties.bootstrap.servers' = 'redpanda:29092',
    'format' = 'json',
    'properties.group.id' = 'opensky-grp-1'
)
```
15) Finally we can run the queries on the data, below are few examples

```
simple select on data
"select id , originCountry from opensky;"
```
