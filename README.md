# PROJECT: PySpark-on-Kubernetes-Word-Count-PageRank

## Table of Contents
- [Introduction](#introduction)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)

## Introduction
This project demonstrates the implementation of Word Count and PageRank algorithms using PySpark on Apache Spark running on Kubernetes. The goal is to leverage the scalability and distributed processing power of Apache Spark and Kubernetes for big data processing.

## Features
- Implement Word Count using PySpark.
- Implement PageRank using PySpark.

## Prerequisites
- Google Cloud SDK (for GKE)
- Kubernetes cluster (GKE or local with Minikube)
- kubectl installed and configured
- Helm installed
- Docker installed
- PySpark installed

## Installation

### Setting up Kubernetes and Apache Spark

1. **Create a Cluster on GKE**:
    ```sh
    gcloud container clusters create spark --num-nodes=1 --machine-type=e2-highmem-2 --region=us-west1 --disk-type=pd-standard
    ```

2. **Install the NFS Server Provisioner**:
    ```sh
    helm repo add stable https://charts.helm.sh/stable

    helm install nfs stable/nfs-server-provisioner \
        --set persistence.enabled=true,persistence.size=5Gi
    ```

3. **Create a Persistent Disk Volume and a Pod to use NFS**:
    ```yaml
    # spark-pvc.yaml
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: spark-data-pvc
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 2Gi
      storageClassName: nfs
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: spark-data-pod
    spec:
      volumes:
        - name: spark-data-pv
          persistentVolumeClaim:
            claimName: spark-data-pvc
      containers:
        - name: inspector
          image: bitnami/minideb
          command:
            - sleep
            - infinity
          volumeMounts:
            - mountPath: "/data"
              name: spark-data-pv
    ```

    Apply the YAML descriptor:
    ```sh
    kubectl apply -f spark-pvc.yaml
    ```

4. **Create and Prepare Your Application JAR File**:
    ```sh
    docker run -v /tmp:/tmp -it bitnami/spark -- find /opt/bitnami/spark/examples/jars/ -name spark-examples* -exec cp {} /tmp/my.jar \;
    ```

5. **Add a Test File for Word Count**:
    ```sh
    echo "how much wood could a woodpecker chuck if a woodpecker could chuck wood" > /tmp/test.txt
    ```

6. **Copy the JAR File and Test File to the PVC**:
    ```sh
    kubectl cp /tmp/my.jar spark-data-pod:/data/my.jar
    kubectl cp /tmp/test.txt spark-data-pod:/data/test.txt
    ```

    Verify the files inside the persistent volume:
    ```sh
    kubectl exec -it spark-data-pod -- ls -al /data
    ```

7. **Deploy Apache Spark on Kubernetes Using the Shared Volume**:
    ```yaml
    # spark-chart.yaml
    service:
      type: LoadBalancer
    worker:
      replicaCount: 3
      extraVolumes:
        - name: spark-data
          persistentVolumeClaim:
            claimName: spark-data-pvc
      extraVolumeMounts:
        - name: spark-data
          mountPath: /data
    ```

    Deploy using Helm:
    ```sh
    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm install spark bitnami/spark -f spark-chart.yaml
    ```

8. **Get the External IP of the Running Pod**:
    ```sh
    kubectl get svc -l "app.kubernetes.io/instance=spark,app.kubernetes.io/name=spark"
    ```

    Open the external IP in your browser:
    ```
    http://<EXTERNAL_IP>
    ```

## Usage

### Word Count on Spark

1. **Submit a Word Count Task**:
    ```sh
    kubectl exec -it spark-master-0 -- spark-submit --master spark://<<EXTERNAL_IP>>:7077 --deploy-mode cluster --class org.apache.spark.examples.JavaWordCount /data/my.jar /data/test.txt![image](https://github.com/zeynnepps/PySpark-on-Kubernetes-Word-Count-PageRank/assets/49025266/46ac55d7-eda5-409d-8a8d-1a44f607602e)
    ```

2. **View the Output of the Completed Jobs**:
    - Get the worker node IP address of the finished task from the browser.
    - Get the name of the worker node:
      ```sh
      kubectl get pods -o wide | grep <WORKER_NODE_ADDRESS>
      ```

    - Execute the pod and see the result:
      ```sh
      kubectl exec -it <worker-node-name> -- bash
      cd /opt/bitnami/spark/work
      cat driver-<timestamp>/stdout
      ```

### Running PageRank on PySpark

1. **Execute the Spark Master Pod**:
    ```sh
    kubectl exec -it spark-master-0 -- bash
    ```

2. **Run PySpark**:
    ```sh
    pyspark
    ```

3. **Exit PySpark**:
    ```sh
    exit()
    ```

4. **Navigate to the PageRank Script Directory**:
    ```sh
    cd /opt/bitnami/spark/examples/src/main/python
    ```

5. **Run PageRank Using PySpark**:
    ```sh
    spark-submit pagerank.py /opt 2
    ```
