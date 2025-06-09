## SETUP STRIMZI KAFKA CLUSTER ON LOCAL USING K3S FROM SCRATCH USING NODEPORT , PVC & HELM ##

# Setup k3s on local ubuntu : 
------------------------------------------------------------
# input: 
![image](https://github.com/user-attachments/assets/e546286d-39c4-4062-96ff-8f7b0a759e0d)

curl -sfL https://get.k3s.io | sh - 
# Check for Ready node, takes ~30 seconds 
sudo k3s kubectl get node 

-------------------------------------------------------------
# Output: 
kubectl get nodes

![image](https://github.com/user-attachments/assets/1b7708a8-029d-48ea-95b9-524cd894063f)


--------------------------------------------------------------------------------------------------

# Create Kafka namespace first 
kubectl create ns kafka 

#  Add helm repo  
helm repo add strimzi http://strimzi.io/charts/

# deploy operator using helm
helm install my-relese strimzi/strimzi-kafka-operator --version 0.36.1 -n kafka

# check its status
![image](https://github.com/user-attachments/assets/f4ea0f85-da4c-4748-84ad-f25dedb05440)


----------------------------------------------------------------------------------------------------

# Create kafka-cluster ( 1 Broker , 1 Zookeeper, PVC 5GB each , Kafka-exporter , entity-operator, kafka-metrix configmap ) 
# input: 
kubectl apply -f kakfa.yaml -n kafka 


# Output: 
![image](https://github.com/user-attachments/assets/d3293459-5614-4a65-bbf7-87fe7cc0c7e5)

----------------------------------------------------------------------------------------------------------------------------------------------------------

# Create kafkatopic using KafkaTopic CR  ( my-topic ) 
kubectl create -f kafka-topic.yaml -n kafka 

# Verify Topic is Created ( my-topic )
# Input:
kubectl get kafkatopic -n kafka

# Output: 
![image](https://github.com/user-attachments/assets/7d86e487-8dd6-4b95-89b2-2601f7748400)

-------------------------------------------------------------------------
# Create Prducer to produce message in the topic 
kubectl create -f producer.yaml -n kafka

-------------------------------------------------------------------------
# create consumer deployemnt in the cluster 
kubectl create -f consumer.yaml -n kafka

-------------------------------------------------------------------------
# Verify both deployed in the cluster 
kubectl get deploy -n kafka

# Output
![image](https://github.com/user-attachments/assets/8fb0e027-c02f-400f-8993-9f8cc16cd07c)


----------------------------------------------------------------------------
# verify producer logs 
# input
kubectl -n kafka logs deploy/kafka-producer -f


# Output
![image](https://github.com/user-attachments/assets/988743ba-a9d4-4e31-8125-386b222a8b6c)


--------------------------------------------------------------------------------------------
# verify Consumer logs
kubectl -n kafka logs deploy/kafka-consumer -f

# Output: 
![image](https://github.com/user-attachments/assets/f3b2d0cf-944e-4d15-a69d-86f97ce8488e)


--------------------------------------------------------------------------------------------
# Check message from broker from the same topic 
kubectl exec -n kafka -it my-kafka-cluster-kafka-0 --   bin/kafka-console-consumer.sh   --bootstrap-server localhost:9092   --topic my-topic

# Output
![image](https://github.com/user-attachments/assets/93c41e13-e631-45a7-bb15-23a4be130e82)

----------------------------------------------------------------------------------------------
# Verify Service Exposed to Nodeport 
kubectl get svc -n kafka

# Output
![image](https://github.com/user-attachments/assets/1745524c-5cc5-4a48-8951-06e1a7bc04fa)

-----------------------------------------------------------------------------------------------
# Verify PVC is used 
kubectl get pvc -n kafka ( 5GB  ) 

# Output
![image](https://github.com/user-attachments/assets/bd492544-1e65-44bf-962e-28c909cdf152)


---------------------------------------------------------------------------------------------------
# Connect to kafka broker using external address of node 
kubectl exec -n kafka -it my-kafka-cluster-kafka-0 --   bin/kafka-console-consumer.sh   --bootstrap-server 172.25.23.81:30401   --topic my-topic

# Output
![image](https://github.com/user-attachments/assets/17bd0d02-1aa8-4ee7-a25a-596d2c6f3aad)

