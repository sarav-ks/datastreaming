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

Update the plugin.path to the location of the build output from step #4 above
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
7) Create single node RedPanda Kafka broker as shown below

```
docker run -d --pull=always --name=redpanda-1 --rm \
-p 8081:8081 \
-p 8082:8082 \
-p 9092:9092 \
-p 9644:9644 \
docker.redpanda.com/vectorized/redpanda:latest \
redpanda start \
--overprovisioned \
--seeds "redpanda-1:33145" \
--set redpanda.empty_seed_starts_cluster=false \
--smp 1  \
--memory 1G \
--reserve-memory 0M \
--check=false \
--advertise-rpc-addr redpanda-1:33145
```
