---
layout: post
title: Pengenalan ScyllaDB
date: 2023-06-26
tags: nosql scylladb
excerpt_separator: <!--more-->
---

Dalam artikel ini akan dibahas sekilas mengenai ScyllaDB dan jargon-jargon yang sering digunakan di dalam dokumentasi.
Untuk pendalaman materi lebih lanjut, dapat membaca dokumentasi dan / atau mengikuti Lab.

<!--more-->

## 1 Apa itu ScyllaDB

ScyllaDB digolongkan sebagai [NoSQL](https://en.wikipedia.org/wiki/NoSQL) dengan sub-kategori [Wide Column](https://en.wikipedia.org/wiki/Wide-column_store).
Turunan dari [Apache Cassandra](https://cassandra.apache.org/) namun ditulis ulang dengan bahasa C++ dengan tujuan mencapai throughput (kuantitas aliran data) lebih tinggi dengan latency (response time) lebih rendah.

Dalam peng-kategori-an [CAP Thorem](https://en.wikipedia.org/wiki/CAP_theorem), ScyllaDB masuk dalam kriteria AP (Availability + Partition Tolerance), dimana untuk Consistency, ekspektasi-nya adalah [Eventual Consistency](https://en.wikipedia.org/wiki/Eventual_consistency).
Sebagai perbandingan, Relational Database (PostgreSQL, MySQL, Oracle, dll) condong dalam kriteria CA (Consistency + Availability), dan database NoSQL dengan arsitektur master-slave condong dalam kriteria CP (Consistency + Partition Tolerance)

## 2 Arsitektur ScyllaDB

### 2.1 Node {#node}

Satuan terkecil deployment ScyllaDB yang berjalan di atas mesin komputer.
Setiap node mengandung hanya sebahagian dari keseluruhan konten database [cluster](#cluster-node-ring) dan terdiri dari beberapa [shard](#shard).
Node dapat berupa physical machine ataupun virtual machine.
ScyllaDB memerlukan CPU Intel/AMD yang mendukung instruksi [SSE4.2](https://en.wikipedia.org/wiki/SSE4#SSE4.2).

### 2.2 Keyspace dan Table {#keyspace-table}

**Table** adalah struktur di ScyllaDB yang menyimpan data, dapat dipandang sebagai himpunan rows dan columns.

**Keyspace** adalah top-level container yang menghimpun table-table dengan atribut-atribut seragam yang mendefinisikan
bagaimana data di-replikasi antar node.

### 2.3 Cluster -- Node Ring {#cluster-node-ring}

![Ilustrasi ScyllaDB Node Ring](/images/scylla-node-ring.jpg)


Multi node ScyllaDB di-himpun sebagai cluster, dengan bentukan logikal-nya serupa ring, didasarkan atas hash ring.
Bentukan logikal ring ini adalah bagian dari design untuk mendistribusikan data antar node.
Data di-replikasi secara automatis tergantung dari nilai [Replication Factor](#replikasi-data).

Jumlah minimum node untuk membentuk cluster adalah 3 (tiga) node.
Jumlah node dalam cluster dapat berubah sewaktu-waktu, baik bertambah (karena penambahan storage ataupun CPU)
ataupun berkurang (karena system maintenance ataupun failure).
Ketika hal ini terjadi, cluster ScyllaDB akan melakukan re-konfigurasi dan balancing data secara otomatis.

Setiap node dalam cluster sifatnya setara, namun selalu harus ada minimal satu node yang menjadi _seed node_.
Seed Node diperlukan sebagai gerbang koneksi awal node / client, yang kemudian setelah meta informasi cluster
dibagikan dan sync, seed node tidak lagi diperlukan.

### 2.4 Replikasi Data {#replikasi-data}

Untuk mencegah kehilangan data dalam cluster ketika salah satu node hilang, data selalu di-replikasi di beberapa node.
Jumlah duplikasi data dalam cluster ini disebut **Replication Factor (RF)**.
Nilai **RF** ini didefinisikan ketika pembentukan **Keyspace**.

### 2.5 Shard {#shard}

Setiap node ScyllaDB terdiri atas beberapa shard, yang masing-masing memegang kendali atas satu porsi dari total keseluruhan data cluster.
ScyllaDB membuat shard sebanyak jumlah vCPU / hyperthread.
Setiap shard dioperasikan dengan basis [shared-nothing architecture](https://en.wikipedia.org/wiki/Shared-nothing_architecture) dengam meng-alokasi-kan secara terpisah untuk RAM dan storage, menghindarkan locking antar shard.

### 2.6 Datacenter {#datacenter}

![Ilustrasi ScyllaDB Datacenter](/images/scylla-datacenter.jpg)

**Datacenter** adalah fasilitas fisik yang menaungi servers, storage dan peralatan network.

**Rack** adalah frame metal yang menampung perangkat keras seperti server, hard disk, peralatan network dan peralatan network lainnya.

ScyllaDB dirancang untuk dapat melibatkan topologi datacenter.
Sebagai contoh, [replica nodes](#replikasi-data) dapat ditempatkan pada rack yang berbeda dalam satu datacenter untuk meminimalkan resiko kegagalan jaringan ataupun masalah kelistrikan.
Mekanisme ScyllaDB dalam memfasilitasi masalah ini adalah [snitches](#snitches).

ScyllaDB juga menawarkan multi-datacenter replikasi.
Untuk mendefinisikan berapa banyak datacenter yang terlibat dalam proses replikasi dan Replication Factor per masing-masing datacenter dibahas pada [Strategi Replikasi](#strategi-replikasi).

### 2.7 Strategi Replikasi {#strategi-replikasi}

Strategi ini menentukan di node mana suatu replika ditempatkan.
Saat ini ScyllaDB memiliki 2 (dua) mode untuk strategi ini:

1. **SimpleStrategy** -- Strategi ini cocok untuk mode development, dimana topologi network hanya satu datacenter dan satu rack.
Strategi ini tidak dianjurkan untuk digunakan di mode production.
2. **Network Topology Strategy** -- Strategi ini memperhitungkan topologi network, dengan mendefinisikan jumlah replikasi untuk tiap datacenter.

### 2.8 Token Ranges {#token-ranges}

![Ilustrasi ScyllaDB Ring Token](/images/scylla-ring-token.jpg)

Setiap node di dalam ring di beri alokasi range.
Setiap data yang masuk di-mapping ke token tertentu menggunakan hash dari partition key.

### 2.9 Consistency Level (CL) {#consistency-level}

Consistency Level (CL) adalah jumlah node yang meng-acknowledge operasi read/write sebelum dinyatakan sukses.

Untuk suatu operasi read/write dinyatakan sukses dari sisi client, sejumlah node harus menyepakati state data terlibat.
Tiap-tiap operasi read/write di ScyllaDB dapat mendefinisikan sendiri nilai CL yang berbeda.

1. `CL ONE`: Cukup mendapat respon dari satu node replika.
1. `CL ALL`: Harus mendapatkan respon dari semua node replika.
1. `CL LOCAL_QUORUM`: Cukup mendapatkan respon dari quorum (`1/2 jumlah node replika + 1`).
1. `CL EACH_QUORUM`: Hanya pada skenario multi-datacenter, untuk semua DC harus mencapai LOCAL_QUORUM.
Mode ini tidak berlaku untuk operasi read.

ScyllaDB menjamin konsistensi data di cluster sebagai **eventually consistency**, karena dengan kombinasi-kombinasi CL ini, memungkinkan pada satu saat data yang dibaca oleh client belum merupakan data yang terakhir ditulis (karena node replika yang me-respon berbeda), namun pada akhirnya akan konsisten di seluruh replika.

![Ilustrasi ScyllaDB operasi write dengan CL ONE dan RF=3](/images/scylla-write_cl_one_rf_three.jpg)

Ilustrasi di atas menggambarkan operasi write dengan mode `CL ONE` pada table dengan `RF 3`.
Panah ganda menunjukkan request dan response proses write.
Karena mode `CL ONE`, koordinator (`V`), hanya perlu menunggu response dari salah satu node replika (dalam kasus di atas, node `W`) sebelum mengembalikan sukses ke client.
Karena `RF=3`, operasi itu juga dilakukan di node `X` dan `Z`, tetapi koordinator tidak perlu menunggu untuk mengembalikan sukses ke client, koordinator tetap akan menerima respon mereka untuk menandai operasi sukses di semua node replika.

![Ilustrasi ScyllaDB operasi write dengan CL QUORUM dan RF=3](/images/scylla-write_cl_quorum_rf_three.jpg)

Sama seperti kasus sebelumnya, dengan perbedaan di sini koordinator perlu menunggu respon dari 2 node replika.

### 2.10 Gossip {#gossip}

ScyllaDB menggunakan protokol komunikasi inter-node secara desentralisasi dan berkala yang disebut **Gossip**.
Gossip digunakan untuk meng-komunikasikan metadata cluster dan pencarian node peer.
Setiap node berkomunikasi dengan tiga node lainnya.

### 2.11 Snitch {#snitch}

Snitch adalah mekanisme ScyllaDB untuk menentukan rack dan datacentre target pada operasi read/write.
