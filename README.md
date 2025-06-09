############################################################## SETUP STRIMZI KAFKA CLUSTER ON LOCAL USING K3S FROM SCRATCH USING NODEPORT , PVC & HELM ######################################################################

# Setup k3s on local ubuntu : 
------------------------------------------------------------
# input: 

curl -sfL https://get.k3s.io | sh - 
# Check for Ready node, takes ~30 seconds 
sudo k3s kubectl get node 

-------------------------------------------------------------
# Output: 

root@Mr-Worthy:~# kubectl get nodes
NAME        STATUS   ROLES                  AGE    VERSION
mr-worthy   Ready    control-plane,master   2d2h   v1.32.5+k3s1

--------------------------------------------------------------------------------------------------

# Create Kafka namespace first 
kubectl create ns kafka 

#  Add helm repo  
helm repo add strimzi http://strimzi.io/charts/

# deploy operator using helm
helm install my-relese strimzi/strimzi-kafka-operator --version 0.36.1 -n kafka

# check its status

input:
kubectl get pods -n kafka -w

Output:
root@Mr-Worthy:~/kafka# kubectl get pods -n kafka -w
NAME                                        READY   STATUS    RESTARTS   AGE
strimzi-cluster-operator-7db455b9c8-4rnt8   0/1     Running   0          4s
strimzi-cluster-operator-7db455b9c8-4rnt8   1/1     Running   0          31s
----------------------------------------------------------------------------------------------------

# Create kafka-cluster ( 1 Broker , 1 Zookeeper, PVC 5GB each , Kafka-exporter , entity-operator, kafka-metrix configmap ) 
# input: 
kubectl apply -f kakfa.yaml -n kafka 

# verify its deployed properly
# input 
kubectl get pods -n kafka -w

# Output: 
kubectl get all -n kafka
NAME                                                    READY   STATUS    RESTARTS        AGE
pod/my-kafka-cluster-entity-operator-5db4855594-7jsjj   3/3     Running   0               2m58s
pod/my-kafka-cluster-kafka-0                            1/1     Running   0               3m44s
pod/my-kafka-cluster-kafka-exporter-5f979697bb-tcjdm    1/1     Running   1 (2m27s ago)   2m28s
pod/my-kafka-cluster-zookeeper-0                        1/1     Running   0               4m12s
pod/strimzi-cluster-operator-7db455b9c8-4rnt8           1/1     Running   0               6m4s

NAME                                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
service/my-kafka-cluster-kafka-bootstrap            ClusterIP   10.43.120.89    <none>        9091/TCP,9092/TCP                     3m44s
service/my-kafka-cluster-kafka-brokers              ClusterIP   None            <none>        9090/TCP,9091/TCP,8443/TCP,9092/TCP   3m44s
service/my-kafka-cluster-kafka-external-0           NodePort    10.43.142.215   <none>        9093:30401/TCP                        3m44s
service/my-kafka-cluster-kafka-external-bootstrap   NodePort    10.43.202.206   <none>        9093:31627/TCP                        3m44s
service/my-kafka-cluster-zookeeper-client           ClusterIP   10.43.88.221    <none>        2181/TCP                              4m13s
service/my-kafka-cluster-zookeeper-nodes            ClusterIP   None            <none>        2181/TCP,2888/TCP,3888/TCP            4m13s

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-kafka-cluster-entity-operator   1/1     1            1           2m58s
deployment.apps/my-kafka-cluster-kafka-exporter    1/1     1            1           2m28s
deployment.apps/strimzi-cluster-operator           1/1     1            1           6m4s

NAME                                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/my-kafka-cluster-entity-operator-5db4855594   1         1         1       2m58s
replicaset.apps/my-kafka-cluster-kafka-exporter-5f979697bb    1         1         1       2m28s
replicaset.apps/strimzi-cluster-operator-7db455b9c8           1         1         1       6m4s
----------------------------------------------------------------------------------------------------------------------------------------------------------

# Create kafkatopic using KafkaTopic CR  ( my-topic ) 
kubectl create -f kafka-topic.yaml -n kafka 

# Verify Topic is Created ( my-topic )
# Input:
kubectl get kafkatopic -n kafka

# Output: 
root@Mr-Worthy:~/kafka# kubectl get kafkatopic -n kafka
NAME                                                                                               CLUSTER            PARTITIONS   REPLICATION FACTOR   READY
consumer-offsets---84e7a678d08f4bd226872e5cdd4eb527fadc1c6a                                        my-kafka-cluster   50           1                    True
my-topic                                                                                           my-kafka-cluster   1            1                    True
strimzi-store-topic---effb8e3e057afce1ecf67c3f5d8e4e3ff177fc55                                     my-kafka-cluster   1            1                    True
strimzi-topic-operator-kstreams-topic-store-changelog---b75e702040b99be8a9263134de3507fc0cc4017b   my-kafka-cluster   1            1                    True
-------------------------------------------------------------------------
# Create Prducer to produce message in the topic 
kubectl create -f producer.yaml -n kafka

-------------------------------------------------------------------------
# create consumer deployemnt in the cluster 
kubectl create -f consumer.yaml -n kafka

-------------------------------------------------------------------------
# Verify both deployed in the cluster 
# input 
kubectl get deploy -n kafka

# Output
root@Mr-Worthy:~# kubectl get deploy -n kafka
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
kafka-consumer                     1/1     1            1           4h40m
kafka-producer                     1/1     1            0           3h1m
my-kafka-cluster-entity-operator   1/1     1            1           4h50m
my-kafka-cluster-kafka-exporter    1/1     1            1           4h50m
strimzi-cluster-operator           1/1     1            1           4h53m

----------------------------------------------------------------------------
# verify producer logs 
# input
kubectl -n kafka logs deploy/kafka-producer -f


# Output
Collecting kafka-python
  Downloading kafka_python-2.2.11-py2.py3-none-any.whl (309 kB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 309.6/309.6 kB 240.9 kB/s eta 0:00:00
Installing collected packages: kafka-python
Successfully installed kafka-python-2.2.11
WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv

[notice] A new release of pip is available: 23.0.1 -> 25.1.1
[notice] To update, run: pip install --upgrade pip
Sent: Message 0 at 2025-06-09 08:48:43
Sent: Message 1 at 2025-06-09 08:48:46
Sent: Message 2 at 2025-06-09 08:48:51
Sent: Message 3 at 2025-06-09 08:48:56
Sent: Message 4 at 2025-06-09 08:49:01
Sent: Message 5 at 2025-06-09 08:49:06
Sent: Message 6 at 2025-06-09 08:49:11
Sent: Message 7 at 2025-06-09 08:49:16
Sent: Message 8 at 2025-06-09 08:49:18
Sent: Message 9 at 2025-06-09 08:49:23
Sent: Message 10 at 2025-06-09 08:49:28
Sent: Message 11 at 2025-06-09 08:49:33
Sent: Message 12 at 2025-06-09 08:49:38
Sent: Message 13 at 2025-06-09 08:49:43
Sent: Message 14 at 2025-06-09 08:49:45
Sent: Message 15 at 2025-06-09 08:49:50
Sent: Message 16 at 2025-06-09 08:49:55
Sent: Message 17 at 2025-06-09 08:50:00
Sent: Message 18 at 2025-06-09 08:50:05
Sent: Message 19 at 2025-06-09 08:50:10
Sent: Message 20 at 2025-06-09 08:50:15
Sent: Message 21 at 2025-06-09 08:50:17

--------------------------------------------------------------------------------------------
# verify Consumer logs
# input
kubectl -n kafka logs deploy/kafka-consumer -f

# Output: 
Received: Message 0 at 2025-06-09 08:33:11
Received: Message 1 at 2025-06-09 08:33:17
Received: Message 2 at 2025-06-09 08:33:22
Received: Message 3 at 2025-06-09 08:33:27
Received: Message 4 at 2025-06-09 08:33:29
Received: Message 5 at 2025-06-09 08:33:34
Received: Message 6 at 2025-06-09 08:33:39
Received: Message 7 at 2025-06-09 08:33:44
Received: Message 8 at 2025-06-09 08:33:49
Received: Message 9 at 2025-06-09 08:33:54
Received: Message 10 at 2025-06-09 08:33:59
Received: Message 11 at 2025-06-09 08:34:01
Received: Message 12 at 2025-06-09 08:34:06
Received: Message 13 at 2025-06-09 08:34:11
Received: Message 14 at 2025-06-09 08:34:16
Received: Message 15 at 2025-06-09 08:34:21
Received: Message 16 at 2025-06-09 08:34:26
Received: Message 17 at 2025-06-09 08:34:28
Received: Message 18 at 2025-06-09 08:34:33
Received: Message 19 at 2025-06-09 08:34:38
Received: Message 20 at 2025-06-09 08:34:43
Received: Message 21 at 2025-06-09 08:34:48
Received: Message 22 at 2025-06-09 08:34:53
Received: Message 23 at 2025-06-09 08:34:58
Received: Message 24 at 2025-06-09 08:35:01
Received: Message 25 at 2025-06-09 08:35:06

--------------------------------------------------------------------------------------------
# Check message from broker from the same topic 
# input 
kubectl exec -n kafka -it my-kafka-cluster-kafka-0 --   bin/kafka-console-consumer.sh   --bootstrap-server localhost:9092   --topic my-topic

# Output
Defaulted container "kafka" out of: kafka, kafka-init (init)
Message 2246 at 2025-06-09 11:54:53
Message 2247 at 2025-06-09 11:54:58
Message 2248 at 2025-06-09 11:55:03
Message 2249 at 2025-06-09 11:55:08
Message 2250 at 2025-06-09 11:55:13
----------------------------------------------------------------------------------------------
# Verify Service Exposed to Nodeport 
# Input 
kubectl get svc -n kafka

# Output
root@Mr-Worthy:~/kafka# kubectl get svc -n kafka
NAME                                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
my-kafka-cluster-kafka-bootstrap            ClusterIP   10.43.120.89    <none>        9091/TCP,9092/TCP                     4h58m
my-kafka-cluster-kafka-brokers              ClusterIP   None            <none>        9090/TCP,9091/TCP,8443/TCP,9092/TCP   4h58m
my-kafka-cluster-kafka-external-0           NodePort    10.43.142.215   <none>        9093:30401/TCP                        4h58m
my-kafka-cluster-kafka-external-bootstrap   NodePort    10.43.202.206   <none>        9093:31627/TCP                        4h58m
my-kafka-cluster-zookeeper-client           ClusterIP   10.43.88.221    <none>        2181/TCP                              4h58m
my-kafka-cluster-zookeeper-nodes            ClusterIP   None            <none>        2181/TCP,2888/TCP,3888/TCP            4h58m

-----------------------------------------------------------------------------------------------
# Verify PVC is used 
# Input
kubectl get pvc -n kafka ( 5GB  ) 

# Output
root@Mr-Worthy:~/kafka# kubectl get pvc -n kafka
NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
data-0-my-kafka-cluster-kafka-0     Bound    pvc-0741296c-c845-4df3-a077-f1b2f74beace   5Gi        RWO            local-path     <unset>                 4h59m
data-my-kafka-cluster-zookeeper-0   Bound    pvc-053e1c40-871b-45ff-8104-05380dc19d93   5Gi        RWO            local-path     <unset>                 4h59m

---------------------------------------------------------------------------------------------------
