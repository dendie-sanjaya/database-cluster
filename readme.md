Database clustering is a technique to group several database servers into one unit that works together to improve availability, scalability, and reliability. With clustering, data can be distributed and replicated automatically between nodes, so if one node fails, other nodes can still serve requests without interruption.

Sharding and Replication are two fundamental pillars in modern distributed database architecture that complement each other to achieve high scalability and data availability.

Sharding focuses on splitting data horizontally, breaking large datasets into several servers to distribute write load and storage.

Replication focuses on copying the same data from one server to another to ensure fault tolerance, high availability, and handling read load balancing.

In this project, we use a distributed database with Elastic.
Elastic (Elasticsearch) is a popular distributed search and analytics engine for storing, searching, and analyzing large amounts of data in real-time. Elastic is used for log management, data analysis, full-text search, and observability.



- [1. Cara deployment menggunakan docker-compose](#1-cara-deployment-menggunakan-docker-compose)
- [2. Aristekur Elastic Cluster](#2-aristekur-elastic-cluster)
	- [2.1. Kibana \& Apps (Layer Pengguna)](#21-kibana--apps-layer-pengguna)
	- [2.2. HAProxy (Load Balancer)](#22-haproxy-load-balancer)
	- [2.3. Elastic Clustering (Core Layer)](#23-elastic-clustering-core-layer)
	- [2.4. Storage 1 \& Storage 2 (Data Layer)](#24-storage-1--storage-2-data-layer)
- [3. Level Penerapan Sharding](#3-level-penerapan-sharding)
- [4. Distribution Data Skema 2 Node / 2 Server  (1 Share \& 1 Replication)](#4-distribution-data-skema-2-node--2-server--1-share--1-replication)
	- [4.1 Distribusi Data Write \& Read (Load Balance)](#41-distribusi-data-write--read-load-balance)
	- [4.2  Auto Fail Over](#42--auto-fail-over)
	- [4.3  Skema After Recover](#43--skema-after-recover)
		- [4.3  Cara Setting](#43--cara-setting)
- [5. Distribution Data Skema 2 Node / 2 Server  (2 Share \& 1 Replication)](#5-distribution-data-skema-2-node--2-server--2-share--1-replication)
	- [5.1 Distribusi Data Write \& Read (Load Balance)](#51-distribusi-data-write--read-load-balance)
	- [5.2  Auto Fail Over](#52--auto-fail-over)
	- [5.3  Skema After Recover](#53--skema-after-recover)
	- [5.4 Cara Setting Index Shard dan Replication di Elastic](#54-cara-setting-index-shard-dan-replication-di-elastic)
	- [5.5  Cara Reroute](#55--cara-reroute)
- [6. Distribution Data Skema 3 Node / 3 Server  (3 Share \& 1 Replication)](#6-distribution-data-skema-3-node--3-server--3-share--1-replication)
- [7. Cek setting shard dan replica pada index:](#7-cek-setting-shard-dan-replica-pada-index)
- [8. Cek Distribusi](#8-cek-distribusi)
- [9. Penting tidak bisa mengubah jumlah shard](#9-penting-tidak-bisa-mengubah-jumlah-shard)



## 1. Cara deployment menggunakan docker-compose

Cara deployment menggunakan docker-compose
```
docker-compose up -d
```
![Screen Shoot](./ss/docker-1.jpg)

--

![Screen Shoot](./ss/docker-2.jpg)


## 2. Aristekur Elastic Cluster 

![Screen Shoot](./design/arsitektur.jpg)


Berikut adalah rincian alur kerja dan komponennya:

### 2.1. Kibana & Apps (Layer Pengguna)

![Screen Shoot](./ss/kibana.jpg)

![Screen Shoot](./ss/kibana2.jpg)

Kibana adalah antarmuka visualisasi data yang berjalan di atas Elasticsearch. Kibana digunakan untuk membuat dashboard, grafik, dan visualisasi data yang tersimpan di Elastic, sehingga memudahkan analisis dan monitoring data.

Ini adalah antarmuka atau aplikasi yang berinteraksi dengan data.
Kibana: Digunakan untuk visualisasi data, pembuatan dasbor, dan manajemen indeks Elasticsearch.
Apps: Merujuk pada aplikasi pihak ketiga atau buatan sendiri yang mengirimkan kueri (query) untuk mengambil atau memasukkan data ke dalam sistem.

### 2.2. HAProxy (Load Balancer)

HAProxy adalah software load balancer dan proxy yang digunakan untuk mendistribusikan trafik ke beberapa server backend secara efisien. 

HAProxy bertindak sebagai pintu gerbang tunggal (Entry Point).
Tugas Utama: Menerima permintaan dari Kibana/Apps dan mendistribusikannya secara merata ke node-node yang ada di dalam klaster Elasticsearch.
Keuntungan: Jika salah satu node Elasticsearch mati, HAProxy akan secara otomatis mengalihkan trafik ke node yang masih aktif, sehingga sistem tetap berjalan tanpa hambatan bagi pengguna.

### 2.3. Elastic Clustering (Core Layer)
Ini adalah inti dari arsitektur, yang terdiri dari dua node atau lebih yang saling terhubung:
Elastic Node 1 & Node 2: Dua server Elasticsearch yang menjalankan layanan pencarian dan penyimpanan data. Mereka saling berkomunikasi (ditunjukkan oleh panah vertikal) untuk menyinkronkan status klaster dan memastikan replikasi data terjadi.
Dashed Box (Kotak Putus-putus): Menandakan bahwa kedua node ini berada dalam satu kesatuan logis yang disebut Cluster.

### 2.4. Storage 1 & Storage 2 (Data Layer)
Masing-masing node memiliki media penyimpanannya sendiri.
Dalam konfigurasi klaster yang baik, data biasanya direplikasi (disalin) antar storage. Jika Storage 1 rusak, data tetap aman karena ada salinannya di Storage 2 yang diakses melalui Node 2.


## 3. Level Penerapan Sharding 
Penerapan sharding pada umum nya berada di level table
Level Tabel (Paling Sering)	Membagi baris pada satu tabel yang sangat besar dan mendistribusikan ke banyak storage 
Meningkatkan skalabilitas penulisan (write) dan mengurangi beban I/O pada satu server.

## 4. Distribution Data Skema 2 Node / 2 Server  (1 Share & 1 Replication)

![Screen Shoot](./ss/skema-2-node-1-share-1-replication-after-recovery.jpg)

Berikut ini adalah contoh beberapa skema Distribusi Database  

### 4.1 Distribusi Data Write & Read (Load Balance)
Distribusi Beban Baca (Read Scaling): Load Balancer mengarahkan permintaan baca (SELECT) ke semua node Storage 1 dan Storage 2 
Distribisi  penulisan (write) selalu diarahkan ke satu ke Primary yang aktif yaitu di storage 1 yang kemudian di replikasi ke storage 2

### 4.2  Auto Fail Over  
jika salah satu Primary gagal diakses atau rusak (storage 1), sistem dapat beralih (failover) secara otomatis ke instans Replika (di secara otomatis di promosikan menjadi Primary) lain yang merupakan salinan lengkapnya, sehingga data tetap dapat diakses tanpa gangguan.

### 4.3  Skema After Recover 
Apabila node storage sudah di perbaiki, maka storage 1 akan menjadi replikasi dan stroage 2 akan menjadi primary


![Screen Shoot](./ss/skema-2-node-1-share-1-replication.jpg)

 
#### 4.3  Cara Setting 

Untuk membuat index dengan 1 shard dan 1 replica:
```
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

## 5. Distribution Data Skema 2 Node / 2 Server  (2 Share & 1 Replication)

![Screen Shoot](./ss/skema-2-node-2-share-1-replication.jpg)

### 5.1 Distribusi Data Write & Read (Load Balance)
Distribusi Beban Baca (Read Scaling): Load Balancer mengarahkan permintaan baca (SELECT) ke semua node Storage 1 dan Storage 2 
Distribusi  penulisan (write) terbagai dua,   sebagain ke Primary di storage 1 sebanyak 50%  data sebagai segmen 1 data,  dan sebagain  ke Primary di storage 2 sebanyak 50% sebagai segemen 2 data, tujuan dari pemisahan untuk meningkat load balacing atau scaling read data menjadi lebih ringan dan cepat.  
Kemudian setiap data yg ada di Segeman 1 dan dan Segmen 2 di lakukan replikasi dengan cara di silang peyimpanan datanya  

### 5.2  Auto Fail Over  
jika salah satu Primary gagal diakses atau rusak (storage 1), sistem dapat beralih (failover) secara otomatis ke (storage 2) dan Replika (di secara otomatis di promosikan menjadi Primary) yang merupakan salinan lengkapnya darn storage 2, sehingga data tetap dapat diakses tanpa gangguan

### 5.3  Skema After Recover 
Apabila node storage 1 sudah di perbaiki, maka storage 1 akan menjadi semua eplikasi dan stroage 2 akan menjadi primary semua

![Screen Shoot](./ss/skema-2-node-2-share-1-replication-after-recovery.jpg)
 
### 5.4 Cara Setting Index Shard dan Replication di Elastic

Untuk membuat index dengan 2 shard dan 1 replica:
```
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

### 5.5  Cara Reroute 

Untuk mengembalikan route agar ternjadi load balance:
```
PUT /data_product
{
	"settings": {
		"index": {
			"number_of_shards": 1,
			"number_of_replicas": 1
		}
	}![Screen Shoot](./ss/skema-2-node-1-share-1-replication.jpg)
}
```


## 6. Distribution Data Skema 3 Node / 3 Server  (3 Share & 1 Replication)
Distribusi Beban Baca (Read Scaling): Load Balancer mengarahkan permintaan baca (SELECT) ke semua node Storage 1 dan Storage 2 dan  Storage 3
Distribusi  penulisan (write) terbagai tiga,  ke Primary di storage 1 sebanyak 1/3  data sebagai segmen 1 data,  dan sebagain  ke Primary di storage 2 sebanyak 1/3 sebagai segemen 2 data, dan sebagain  ke Primary di storage 3 sebanyak 1/3 sebagai segemen 3. 
Tujuan dari pemisahan untuk meningkat load balacing atau scaling read data menjadi lebih ringan dan cepat.  
Kemudian setiap data yg ada di Segeman 1, Segmen 2, Segmen 3 di lakukan replikasi dengan cara di silang peyimpanan datanya  

![Screen Shoot](./ss/skema-3-node-3-share-1-replication.jpg)

## 7. Cek setting shard dan replica pada index:

```
GET /data_product/_settings
```

## 8. Cek Distribusi

Untuk melihat distribusi shard dan replica di cluster:
```
GET /_cat/shards?v
```

## 9. Penting tidak bisa mengubah jumlah shard
Jika perlu mengubah jumlah primary shards (misalnya, dari 1 ke 2 atau sebalik), harus menggunakan proses Reindexing 
Langkah yang paling umum dan fleksibel adalah menggunakan Reindexing (membuat index baru, lalu memindahkan semua data lama ke index baru tersebut).

