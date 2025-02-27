---
description: Pinot quick start in Kubernetes
---

# Running Pinot in Kubernetes

## 1. Prerequisites

{% hint style="info" %}
This quickstart assumes that you already have a running Kubernetes cluster. Please follow the links below to set up a Kubernetes cluster.

* [Enable Kubernetes on Docker-Desktop](https://docs.docker.com/docker-for-mac/kubernetes/)
* [Install Minikube for local setup](https://kubernetes.io/docs/tasks/tools/install-minikube/) \(make sure to run with enough resources e.g. `minikube start --vm=true --cpus=4 --memory=8g --disk-size=50g)`
* [Setup a Kubernetes Cluster using Amazon Elastic Kubernetes Service \(Amazon EKS\)](public-cloud-examples/aws-quickstart.md)
* [Setup a Kubernetes Cluster using Google Kubernetes Engine \(GKE\)](public-cloud-examples/gcp-quickstart.md)
* [Setup a Kubernetes Cluster using Azure Kubernetes Service \(AKS\)](public-cloud-examples/azure-quickstart.md)
{% endhint %}

## 2. Setting up a Pinot cluster in Kubernetes

Before continuing, please make sure that you've downloaded Apache Pinot. The scripts for the setup in this guide can be found in our open source project on GitHub.

The scripts can be found in the Pinot source at `./pinot/kubernetes/helm`

{% tabs %}
{% tab title="Git clone project source" %}
```bash
# checkout pinot
git clone https://github.com/apache/pinot.git
cd pinot/kubernetes/helm
```
{% endtab %}
{% endtabs %}

### 2.1 Start Pinot with Helm

{% tabs %}
{% tab title="Run Helm with Pre-installed Package" %}
Pinot repo has pre-packaged HelmCharts for Pinot and Presto. Helm Repo index file is [here](https://github.com/apache/pinot/blob/master/kubernetes/helm/index.yaml).

```text
helm repo add pinot https://raw.githubusercontent.com/apache/pinot/master/kubernetes/helm
kubectl create ns pinot-quickstart
helm install pinot pinot/pinot \
    -n pinot-quickstart \
    --set cluster.name=pinot \
    --set server.replicaCount=2
```

**NOTE**: Please specify **StorageClass** based on your cloud vendor. For Pinot Server, please don't mount blob store like AzureFile/GoogleCloudStorage/S3 as the data serving file system.

Only use Amazon EBS/GCP Persistent Disk/Azure Disk style disks.

* For AWS: "**gp2**"
* For GCP: "**pd-ssd**" or "**standard**"
* For Azure: "**AzureDisk**"
* For Docker-Desktop: "**hostpath**"
{% endtab %}

{% tab title="Run Helm Script within Git Repo" %}
### 2.1.1 Update helm dependency

```text
helm dependency update
```

### 2.1.2 Start Pinot with Helm

* For Helm **v2.12.1**

If your Kubernetes cluster is recently provisioned, ensure Helm is initialized by running:

```text
helm init --service-account tiller
```

Then deploy a new HA Pinot cluster using the following command:

```text
helm install --namespace "pinot-quickstart" --name "pinot" pinot
```

* For Helm **v3.0.0**

```text
kubectl create ns pinot-quickstart
helm install -n pinot-quickstart pinot pinot
```

### **2.1.3 Troubleshooting \(For helm v2.12.1\)**

* Error: Please run the below command if encountering the following issue:

```text
Error: could not find tiller.
```

* Resolution:

```text
kubectl -n kube-system delete deployment tiller-deploy
kubectl -n kube-system delete service/tiller-deploy
helm init --service-account tiller
```

* Error: Please run the command below if encountering a permission issue:

`Error: release pinot failed: namespaces "pinot-quickstart" is forbidden: User "system:serviceaccount:kube-system:default" cannot get resource "namespaces" in API group "" in the namespace "pinot-quickstart"`

* Resolution:

```text
kubectl apply -f helm-rbac.yaml
```
{% endtab %}
{% endtabs %}

### **2.2 Check Pinot deployment status**

```text
kubectl get all -n pinot-quickstart
```

## 3. Load data into Pinot using Kafka

### **3.1 Bring up a Kafka cluster for real-time data ingestion**

{% tabs %}
{% tab title="For Helm v3.0.0" %}
```text
helm repo add incubator https://charts.helm.sh/incubator
helm install -n pinot-quickstart kafka incubator/kafka --set replicas=1
```
{% endtab %}

{% tab title="For Helm v2.12.1" %}
```text
helm repo add incubator https://charts.helm.sh/incubator
helm install --namespace "pinot-quickstart"  --name kafka incubator/kafka
```
{% endtab %}
{% endtabs %}

### 3.2 Check Kafka deployment status

```text
kubectl get all -n pinot-quickstart | grep kafka
```

Ensure the Kafka deployment is ready before executing the scripts in the following next steps.

```text
pod/kafka-0                                                 1/1     Running     0          2m
pod/kafka-zookeeper-0                                       1/1     Running     0          10m
pod/kafka-zookeeper-1                                       1/1     Running     0          9m
pod/kafka-zookeeper-2                                       1/1     Running     0          8m
```

### **3.3 Create Kafka topics**

The scripts below will create two Kafka topics for data ingestion:

```text
kubectl -n pinot-quickstart exec kafka-0 -- kafka-topics --zookeeper kafka-zookeeper:2181 --topic flights-realtime --create --partitions 1 --replication-factor 1
kubectl -n pinot-quickstart exec kafka-0 -- kafka-topics --zookeeper kafka-zookeeper:2181 --topic flights-realtime-avro --create --partitions 1 --replication-factor 1
```

### **3.4 Load data into Kafka and create Pinot schema/tables**

The script below will deploy 3 batch jobs.

* Ingest 19492 JSON messages to Kafka topic `flights-realtime`  at a speed of 1 msg/sec
* Ingest 19492 Avro messages to Kafka topic `flights-realtime-avro`  at a speed of 1 msg/sec
* Upload Pinot schema `airlineStats`
* Create Pinot table `airlineStats` to ingest data from JSON encoded Kafka topic `flights-realtime`
* Create Pinot table `airlineStatsAvro`  to ingest data from Avro encoded Kafka topic `flights-realtime-avro`

```text
kubectl apply -f pinot/pinot-realtime-quickstart.yml
```

## 4. Query using Pinot Data Explorer

### 4.1 Pinot Data Explorer

Please use the script below to perform local port-forwarding, which will also open Pinot query console in your default web browser.

This script can be found in the Pinot source at `./pinot/kubernetes/helm/pinot`

```text
./query-pinot-data.sh
```

## 5. Using Superset to query Pinot

### 5.1 Bring up Superset

Open `superset.yaml` file and goto the line showing `storageClass`. And change it based on your cloud vendor. `kubectl get sc` will get you the `storageClass` value for your Kubernetes system. E.g.

* For AWS: "**gp2**"
* For GCP: "**pd-ssd**" or "**standard**"
* For Azure: "**AzureDisk**"
* For Docker-Desktop: "**hostpath**"

Then run:

```text
kubectl apply -f superset.yaml
```

Ensure your cluster is up by running:

```text
kubectl get all -n pinot-quickstart | grep superset
```

### 5.2 \(First time\) Set up Admin account

```text
kubectl exec -it pod/superset-0 -n pinot-quickstart -- bash -c 'flask fab create-admin'
```

### 5.3 \(First time\) Init Superset

```text
kubectl exec -it pod/superset-0 -n pinot-quickstart -- bash -c 'superset db upgrade'
kubectl exec -it pod/superset-0 -n pinot-quickstart -- bash -c 'superset init'
```

### 5.4 Load Demo data source

```text
kubectl exec -it pod/superset-0 -n pinot-quickstart -- bash -c 'superset import_datasources -p /etc/superset/pinot_example_datasource.yaml'
kubectl exec -it pod/superset-0 -n pinot-quickstart -- bash -c 'superset import_dashboards -p /etc/superset/pinot_example_dashboard.json'
```

### 5.5 Access Superset UI

You can run below command to navigate superset in your browser with the previous admin credential.

```text
./open-superset-ui.sh
```

You can open the imported dashboard by clicking `Dashboards` banner and then click on `AirlineStats`.

## 6. Access Pinot using Presto

### 6.1 Deploy Presto using Pinot plugin

You can run the command below to deploy a customized Presto with the Pinot plugin installed.

{% tabs %}
{% tab title="Helm" %}
```text
helm install presto pinot/presto -n pinot-quickstart
```
{% endtab %}

{% tab title="K8s Scripts" %}
```text
kubectl apply -f presto-coordinator.yaml
```
{% endtab %}
{% endtabs %}

The above command deploys Presto with default configs. For customizing your deployment, you can run the below command to get all the configurable values.

```text
helm inspect values pinot/presto > /tmp/presto-values.yaml
```

After modifying the `/tmp/presto-values.yaml` file, you can deploy Presto with:

```text
helm install presto pinot/presto -n pinot-quickstart --values /tmp/presto-values.yaml
```

Once you deployed the Presto, You can check Presto deployment status by:

```text
kubectl get pods -n pinot-quickstart
```

![Sample Output of K8s Deployment Status](https://lh3.googleusercontent.com/t4LnQL4xalac-ObeF37LvtrroHzfgr84lYv3av_MI1NWIcUG1Kuc9uDmJHdYbyJiEfLuBdvT3451VS49lGO_i167m82EM2ZfWk84Zvj-Hib8hHLI8mZt20akpdEh3BLV1Q4ETaL_)

### 6.2 Query Presto using Presto CLI

Once Presto is deployed, you can run the below command from [here](https://github.com/apache/pinot/blob/master/kubernetes/helm/presto/pinot-presto-cli.sh), or just follow steps 6.2.1 to 6.2.3.

```text
./pinot-presto-cli.sh
```

#### **6.2.1 Download Presto CLI**

```text
curl -L https://repo1.maven.org/maven2/com/facebook/presto/presto-cli/0.246/presto-cli-0.246-executable.jar -o /tmp/presto-cli && chmod +x /tmp/presto-cli
```

**6.2.2 Port forward presto-coordinator port 8080 to localhost port 18080**

```text
kubectl port-forward service/presto-coordinator 18080:8080 -n pinot-quickstart> /dev/null &
```

#### **6.2.3 Start Presto CLI with pinot catalog to query it then query it**

```text
/tmp/presto-cli --server localhost:18080 --catalog pinot --schema default
```

6.2.4 Query Pinot data using Presto CLI

![](https://lh3.googleusercontent.com/e5h4FIpvLjSiLi9J2WeABD8CAhtxj-vjyzjgj4pgmtkY-0o3uVr-qNHlOrFV3RGTu8ah4VFtLw5vqAABTATvTjLF5g2E6VnpDKRweAp_akfD7EeabgXVr2ObbUMIsDZ9pibO1a9j)

### 6.3 Sample queries to execute

* List all catalogs

```text
presto:default> show catalogs;
```

```text
 Catalog
---------
 pinot
 system
(2 rows)

Query 20191112_050827_00003_xkm4g, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0:01 [0 rows, 0B] [0 rows/s, 0B/s]
```

* List All tables

```text
presto:default> show tables;
```

```text
    Table
--------------
 airlinestats
(1 row)

Query 20191112_050907_00004_xkm4g, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0:01 [1 rows, 29B] [1 rows/s, 41B/s]
```

* Show schema

```text
presto:default> DESCRIBE pinot.dontcare.airlinestats;
```

```text
        Column        |  Type   | Extra | Comment
----------------------+---------+-------+---------
 flightnum            | integer |       |
 origin               | varchar |       |
 quarter              | integer |       |
 lateaircraftdelay    | integer |       |
 divactualelapsedtime | integer |       |
......

Query 20191112_051021_00005_xkm4g, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0:02 [80 rows, 6.06KB] [35 rows/s, 2.66KB/s]
```

* Count total documents

```text
presto:default> select count(*) as cnt from pinot.dontcare.airlinestats limit 10;
```

```text
 cnt
------
 9745
(1 row)

Query 20191112_051114_00006_xkm4g, FINISHED, 1 node
Splits: 17 total, 17 done (100.00%)
0:00 [1 rows, 8B] [2 rows/s, 19B/s]
```

## 7. Deleting the Pinot cluster in Kubernetes

```text
kubectl delete ns pinot-quickstart
```

