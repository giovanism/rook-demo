# Demonstrasi Rook dan _Dynamic Provisioning_

Hai! Repository ini bertujuan untuk membantu menunjukkan pemakaian _storage
orchestrator_ Rook di ekosistem Kubernetes. Selain itu ada juga contoh sederhana
bagaimana mengonsumsi _storage_ yang dialokasikan menggunakan _dynamic
provisoning_.

## Rook

Kebanyakan langkah-langkah dan _manifest_ yang dipakai di sini diambil dari
dokumentasi resmi [Rook][rook-io] dan panduan _quickstart_
[Ceph Storage][quickstart-ceph].

### _Setup_ Klaster

Berikut bukan batas minimum yang mutlak, tapi setup yang saya gunakan untuk demo
ini:

- Kubernetes v1.18
- 3 _worker_ node
- satu block storage berukuran 8GB untuk setiap node (e.g. /dev/sda) yang masih
  kosong tanpa tabel partisi atau _filesystem_.

Untuk konfigurasi yang lebih lanjut lagi, bisa merujuk ke
[_Prerequisites_][ceph-prerequisites] lengkap untuk membuat klaster Ceph.

### Deploy Operator Rook dan Buat Klaster Ceph

Contoh ini hanya akan mendemonstrasikan `volumeMode: Filesystem`, sehingga kita
hanya membutuhkan CSI driver untuk CephFS saja. Di _manifest_ `operator.yaml`
yang sudah diberikan, CSI driver RBD sudah dimatikan dengan mengubah ConfigMap rook-ceph-operator-config.

```diff
 kind: ConfigMap
 apiVersion: v1
 metadata:
   name: rook-ceph-operator-config
   namespace: rook-ceph
   data:
-    ROOK_CSI_ENABLE_RBD: "true"
+    ROOK_CSI_ENABLE_RBD: "false"
```

Selanjutnya, untuk mendeploy Operator Rook beserta Klaster Ceph hanya perlu
menerapkan berkas-berkas _manifest_ berikut.

```bash
kubectl create -f common.yaml
kubectl create -f operator.yaml
kubectl create -f cluster.yaml
```

Klaster Ceph yang siap digunakan kurang lebih akan memiliki mon yang memenuhi
quorum, sebuah mgr, dan paling tidak satu OSD yang aktif.

```
Every 2,0s: kubectl get -n rook-ceph pods                clustermodulus04: Sun Jul 26 16:19:25 2020

NAME                                              READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-h95tl                            3/3     Running     0          5m33s
csi-cephfsplugin-wxzbq                            3/3     Running     0          5m33s
csi-cephfsplugin-xh68r                            3/3     Running     0          5m39s
rook-ceph-crashcollector-k8s-1-7fbcbd7885-6x4rk   1/1     Running     0          44m
rook-ceph-crashcollector-k8s-2-84748868b4-qgqkj   1/1     Running     0          12m
rook-ceph-crashcollector-k8s-3-88cddfcf9-5qwj4    1/1     Running     0          51m
rook-ceph-mgr-a-8698fbd8c5-2s9vm                  1/1     Running     0          26m
rook-ceph-mon-a-7887d68cbd-zq8h2                  1/1     Running     0          51m
rook-ceph-mon-b-59f8d9cf87-bqfc6                  1/1     Running     0          51m
rook-ceph-mon-d-868c7d5785-2fgrf                  1/1     Running     0          44m
rook-ceph-operator-db86d47f5-s9dfm                1/1     Running     0          7m5s
rook-ceph-osd-0-95d5d4ff7-hq9gf                   1/1     Running     0          13m
rook-ceph-osd-1-d4c46f558-hffn6                   1/1     Running     0          13m
rook-ceph-osd-2-5897d74c54-pfftv                  1/1     Running     0          7m7s
rook-ceph-osd-prepare-k8s-1-t6c8q                 0/1     Completed   0          19m
rook-ceph-osd-prepare-k8s-2-qmv8b                 0/1     Completed   0          19m
rook-ceph-osd-prepare-k8s-3-k4l9v                 0/1     Completed   0          13m
rook-discover-cltcq                               1/1     Running     0          5m57s
rook-discover-g4zmk                               1/1     Running     0          5m57s
rook-discover-mkhwr                               1/1     Running     0          5m57s
```

Bisa juga menggunakan Pod toolbox serbaguna untuk mengecek status klaster dan
melakukan _troubleshooting_.

```bash
kubectl create -f toolbox.yaml
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- ceph status
```

Status klaster akan ditampilkan kurang lebih seperti berikut.

```
  cluster:
    id:     5d445d60-62b5-4200-b1f1-2984c0c7f654
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,d (age 4m)
    mgr: a(active, since 9m)
    osd: 3 osds: 3 up (since 10m), 3 in (since 10m)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   3.0 GiB used, 21 GiB / 24 GiB avail
    pgs:     1 active+clean
```

### Membuat Filesystem

Untuk membuat _filesystem_ di atas klaster Ceph yang sudah dibuat, cukup perlu
mendefinisikan _resource_ CephFilesystem. CephFilesystem bisa mengatur mulai
dari replikasi metadata, kompresi, sampai batas _resource_ yang akan digunakan
oleh metadata server nanti. Kali ini, _manifest_ default yang diberikan sudah
cukup untuk kebutuhan demo ini, jadi bisa langsung dibuat saja.

```bash
kubectl create -f filesystem.yaml
```

### Menyiapkan _Dynamic Provisioning_

> TODO: needs better wording/explanation

Untuk menggunakan _dynamic provisioning_, masih ada pekerjaan rumah yang harus
dikerjakan. Belum ada StorageClass yang bisa dipakai untuk memenuhi permintaan
_storage_ yang dideskripsikan oleh PVC (PersistentVolumeClaim). StorageClass
yang dibuat akan menggunakan CSI CephFS Provisioner yang telah disediakan oleh
Operator Rook sebelumnya. Di balik layar, _provisioner_ tadi diimplementasikan
sebagai _CSI Plugin_.

Buat StorageClass dengan menerapkan _manifest_ `csi-storageclass.yaml`.

```bash
kubectl create -f csi-storageclass.yaml
```

Agar lebih afdal, sebagai satu-satunya StorageClass yang tersedia di klaster
bisa ditandai sebagai kelas _default_. StorageClass _default_ dapat menyediakan
PV (PersistentVolume) kepada PVC yang tidak menspesifikasikan kolom
`storageClassName`.

```bash
kubectl patch -n rook-ceph storageclass rook-cephfs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## _Dynamic Provisioning in Action_

> TODO

[rook-io]: https://rook.io
[quickstart-ceph]: https://rook.io/docs/rook/v1.3/ceph-quickstart.html
[ceph-prerequisites]: https://rook.io/docs/rook/v1.3/ceph-prerequisites.html
