# datastreaming
Apache Flink data streaming using Kafka/RedPanda 



![image](https://user-images.githubusercontent.com/64332344/210473707-00454559-f378-482a-829a-9fdb54ad345f.png)


1) Install Docker Desktop
2) Create single node RedPanda Kafka broker as shown below

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
