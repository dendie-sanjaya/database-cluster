# Table of Contents

1. [Introduction](#introduction)
2. [How to Deploy with Docker Compose](#how-to-deploy-with-docker-compose)
3. [Elastic Cluster Architecture](#elastic-cluster-architecture)
   - [Kibana & Apps](#kibana--apps)
   - [HAProxy (Load Balancer)](#haproxy-load-balancer)
   - [Elastic Clustering (Core Layer)](#elastic-clustering-core-layer)
   - [Storage Layer](#storage-layer)
4. [Sharding Level](#sharding-level)
5. [Data Distribution Schemes](#data-distribution-schemes)
   - [2 Node / 2 Server (1 Shard & 1 Replication)](#2-node--2-server-1-shard--1-replication)
   - [2 Node / 2 Server (2 Shard & 1 Replication)](#2-node--2-server-2-shard--1-replication)
   - [3 Node / 3 Server (3 Shard & 1 Replication)](#3-node--3-server-3-shard--1-replication)
6. [Elastic Index Settings](#elastic-index-settings)
7. [Check Shard and Replica Settings](#check-shard-and-replica-settings)
8. [Important: Changing Shard Number](#important-changing-shard-number)

# Introduction

Database clustering is a way to group several database servers so they work together. This helps make your data more available, scalable, and reliable. If one server fails, others can still handle requests.

Sharding and Replication are two important ideas in distributed databases. Sharding splits big data into smaller parts and puts them on different servers. Replication copies the same data to other servers so your data is safe and always available.

In this project, we use Elastic (Elasticsearch) as a distributed database. Elastic is a popular search and analytics engine for storing, searching, and analyzing large data in real-time. It is used for log management, data analysis, full-text search, and observability.

# How to Deploy with Docker Compose

To deploy with Docker Compose, use this command:

```shell
docker-compose up -d
```

![Screen Shoot](./ss/docker-1.jpg)
![Screen Shoot](./ss/docker-2.jpg)

# Elastic Cluster Architecture

![Screen Shoot](./design/arsitektur.jpg)

## Kibana & Apps

![Screen Shoot](./ss/kibana.jpg)
![Screen Shoot](./ss/kibana2.jpg)

Kibana is a web interface for Elasticsearch. It helps you make dashboards, charts, and visualizations from your data in Elastic. Apps are other programs that send queries to get or add data.

## HAProxy (Load Balancer)

HAProxy is a load balancer and proxy. It sends traffic to different backend servers. If one Elasticsearch node fails, HAProxy will send traffic to the healthy nodes so users are not affected.

## Elastic Clustering (Core Layer)

This is the main part of the architecture. There are two or more Elasticsearch nodes. They talk to each other to keep the cluster status and replicate data. Both nodes are in one logical cluster.

## Storage Layer

Each node has its own storage. In a good cluster setup, data is copied between storages. If one storage fails, the data is still safe in the other storage.

# Sharding Level

Sharding is usually done at the table level.
- Table Level: Split rows in a big table and put them on different storage nodes.
- Purpose: To improve write scalability and reduce load on one server.

# Data Distribution Schemes

## 2 Node / 2 Server (1 Shard & 1 Replication)

![Screen Shoot](./ss/skema-2-node-1-share-1-replication-after-recovery.jpg)

- Read requests are sent to both Storage 1 and Storage 2.
- Write requests go to the active Primary in Storage 1, then are copied to Storage 2.
- If Storage 1 fails, the system switches to the Replica in Storage 2.
- After recovery, Storage 1 becomes the replica and Storage 2 becomes the primary.

![Screen Shoot](./ss/skema-2-node-1-share-1-replication.jpg)

### How to Set Index with 1 Shard & 1 Replica
```json
PUT /data_product
{
  "settings": {
    "index": {
      "number_of_shards": 1,
      "number_of_replicas": 1
    }
  }
}
```

## 2 Node / 2 Server (2 Shard & 1 Replication)

![Screen Shoot](./ss/skema-2-node-2-share-1-replication.jpg)

- Read requests are sent to both Storage 1 and Storage 2.
- Write requests are split: 50% to Primary in Storage 1 (segment 1), 50% to Primary in Storage 2 (segment 2).
- Each segment is replicated to the other storage.
- If Storage 1 fails, the system switches to Storage 2 and its Replica.
- After recovery, Storage 1 becomes all replicas and Storage 2 becomes all primaries.

![Screen Shoot](./ss/skema-2-node-2-share-1-replication-after-recovery.jpg)

### How to Set Index with 2 Shards & 1 Replica
```json
PUT /data_product
{
  "settings": {
    "index": {
      "number_of_shards": 2,
      "number_of_replicas": 1
    }
  }
}
```

### How to Reroute for Load Balance
```json
PUT /data_product
{
  "settings": {
    "index": {
      "number_of_shards": 1,
      "number_of_replicas": 1
    }
  }
}
```

## 3 Node / 3 Server (3 Shard & 1 Replication)

![Screen Shoot](./ss/skema-3-node-3-share-1-replication.jpg)

- Read requests are sent to Storage 1, Storage 2, and Storage 3.
- Write requests are split: 1/3 to Primary in Storage 1 (segment 1), 1/3 to Primary in Storage 2 (segment 2), 1/3 to Primary in Storage 3 (segment 3).
- Each segment is replicated to the other storages.

# Elastic Index Settings

## Check Shard and Replica Settings
```json
GET /data_product/_settings
```

## Check Shard and Replica Distribution
```json
GET /_cat/shards?v
```

# Important: Changing Shard Number

You cannot change the number of primary shards directly. If you need to change the number of primary shards (for example, from 1 to 2), you must use the reindexing process. The usual way is to create a new index and move all old data to the new index.
