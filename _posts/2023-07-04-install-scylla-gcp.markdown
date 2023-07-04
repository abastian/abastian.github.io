---
layout: post
title: Studi Kasus -- Instalasi ScyllaDB di GCP (VM based)
date: 2023-07-04
tags: nosql scylladb gcp
---

Artikel ini akan membahas langkah-langkah instalasi ScyllaDB di cloud platform GCP di atas beberapa VM yang dihimpun sebagai cluster.
Instalasi dilakukan semi-otomatis dengan menggunakan image khusus oleh `scylla-images`

<!--more-->

## 1 Prerequisites {#prerequisites}

* Pastikan semua [port](https://opensource.docs.scylladb.com/stable/operating-scylla/admin.html#cqlsh-networking) telah di-allow.
* Alokasi alamat IP yang akan berpartisipasi dalam cluster.
* Tentukan nama unik sebagai cluster_name (identik untuk semua node).

## 2 Prosedur

Langkah-langkah ini perlu dilakukan untuk setiap node di dalam cluster.

### 2.1 Bangun scylla node.
Node yang akan digunakan ber-tipe n2-highmem-4 (4 vCPU, 32GB memori), 2 internal SSD-NVME (masing-masing 375GB), dengan ScyllaDB opensource v5.2.1
{% highlight bash %}
gcloud compute instances create user-profile-scylla-<n> \
    --project=<project-id> \
    --zone=<zone> \
    --machine-type=n2-highmem-4 \
    --image scylladb-5-2-1 \
    --image-project scylla-images \
    --network-interface=stack-type=IPV4_ONLY,<network-specific> \
    --tags=<firewall-tag> \
    --local-ssd=interface=NVME \
    --local-ssd=interface=NVME
{% endhighlight %}
Gantikan :
* `n` : node-number
* `project-id`: project id dari GCP project
* `zone`: zone di mana node akan di-install
* `network-specific`: konfigurasi network yang diperlukan, termasuk alamat IP yang sudah dialokasikan, konsultasikan dengan team Infra Cloud (lihat [Prerequisites](#prerequisites))
* `firewall-tag`: tag yang didefinisikan ketika setting Firewall, konsultasikan dengan team Infra Cloud (lihat [Prerequisites](#prerequisites))

Setelah node ScyllaDB sukses terbangun, login ke dalam shell dan jalankan `nodetool status`.
Pastikan status node sudah menjadi `UN` (Up + Normal).

### 2.2 Konfigurasi

Sebelum melakukan konfigurasi, pastikan service `scylla-server` sudah dipadamkan terlebih dahulu dengan menjalankan `sudo systemctl stop scylla-server`.
Edit file `/etc/scylla/scylla.yaml`.

* cluster_name - Tetapkan dengan cluster_name yang telah dipilih (lihat [Prerequisites](#prerequisites))
* seeds - Tetapkan dengan IP dari node pertama dan hanya node pertama. Node baru akan menggunakan IP dari seed node ini untuk terhubung ke cluster dan mempelajari topologi dan status cluster
* listen_address - Alamat IP yang digunakan untuk terhubung oleh node Scylla lain di dalam cluster
* endpoint_snitch - `GoogleCloudSnitch`
* rpc_address - Alamat IP yang digunakan untuk terhubung oleh client
* consistent_cluster_management - `true` secara default

#### 2.2.1 (Optional) Set `dc_suffix`

Untuk membedakan DC-DC di dalam satu zone, dapat dilakukan dengan memberikan suffix pada nama DC dengan meng-edit file `/etc/scylla/cassandra-rackdc.properties` dan menambahkan line berikut 
{% highlight yaml %}
dc_suffix=<suffix>
{% endhighlight %}

#### 2.2.2 Restart service scylla-server

Sebelum me-start service scylla server, lakukan pembersihan data terlebih dahulu.

{% highlight bash %}
sudo rm -rf /var/lib/scylla/data
sudo find /var/lib/scylla/commitlog -type f -delete
sudo find /var/lib/scylla/hints -type f -delete
sudo find /var/lib/scylla/view_hints -type f -delete
{% endhighlight %}

{% highlight bash %}
sudo systemctl start scylla-server
{% endhighlight %}

### 2.3 Verifikasi

Setelah semua node di-konfigurasi dan restart, yakinkan di semua node sudah tergabung dengan baik di dalam cluster dengan mengeksekusi `nodetool status` di tiap-tiap node dan status semua node menjadi `UN`.
Pada saat-saat awal, mungkin ada yang masih berstatus `UJ` (Up + Join), hal ini masih normal.

## Sumber Referensi
* [Create a ScyllaDB Cluster - Single Data Center (DC)](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/create-cluster.html)