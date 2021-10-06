# **Cluster Autoscaler EKS**

## Step-01: Penjelasan singkat
- Kubernetes Cluster Autoscaler secara otomatis menyesuaikan jumlah node di cluster Anda saat pod gagal diluncurkan karena kekurangan sumber daya atau saat node di cluster kurang dimanfaatkan dan podnya dapat dijadwal ulang ke node lain di cluster.

## Step-02: Verifikasi apakah NodeGroup ini sebagai --asg-access
- Perlu memastikan bahwa kita memiliki parameter bernama `--asg-access` yang ada selama pembuatan cluster atau nodegroup.
- Verifikasi hal yang sama ketika saya membuat grup node cluster ini

### Menggunakan --asg-access tag dengan tujuan
- Ini memungkinkan IAM policy untuk melakukan cluster-autoscaler
- Meninjau ulang nodegroup IAM role untuk yang sama. 
- Konfigurasi Services -> IAM -> Roles -> eksctl-eksdemo1-nodegroup-XXXXXX
- Klik pada tab **Permissions**
- Lihat inline policy bernama `eksctl-eksdemo1-nodegroup-eksdemo1-ng-private1-PolicyAutoScaling` dalam daftar kebijakan yang terkait dengan peran ini.

## Step-03: Mendeploy Cluster Autoscaler
```
# Deploy the Cluster Autoscaler pada cluster pribadi
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# Tambahkan cluster-autoscaler.kubernetes.io/safe-to-evict anotasi untuk proses deployment
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false"
```
## Step-04: Edit Cluster Autoscaler Deployment untuk menambahkan nama Cluster dan dua parameter lainnya
```
kubectl -n kube-system edit deployment.apps/cluster-autoscaler
```
- **Add cluster name**
```yml
# Sebelum diubah
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<YOUR CLUSTER NAME>

# Setelah diubah
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/eksdemo1
```

- **Tambahkan dua parameter lagi**
```yml
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
```
- **sampel untuk referensi**
```yml
    spec:
      containers:
      - command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/eksdemo1
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
```

## Step-05: Atur Cluster Autoscaler Imager yang terkait dengan versi Cluster EKS dimiliki saat ini
- Buka https://github.com/kubernetes/autoscaler/releases
- Cari versi rilisnya (contoh: 1.16.n) dan memperbarui yang sama. 
- Pastikan Cluster versi dan cluster autoscaler pada versi yang sama
```
# Memperbarui versi Cluster Autoscaler Image
kubectl -n kube-system set image deployment.apps/cluster-autoscaler cluster-autoscaler=us.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler:v1.XY.Z


# Memperbarui versi Cluster Autoscaler Image
kubectl -n kube-system set image deployment.apps/cluster-autoscaler cluster-autoscaler=us.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler:v1.16.5
```

## Step-06: Verifikasi image sudah terimplementasi perubahan yang dilakukan
```
kubectl -n kube-system get deployment.apps/cluster-autoscaler -o yaml
```
- **Contoh keluaran parsial**
```yml
    spec:
      containers:
      - command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/eksdemo1
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
        image: us.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler:v1.16.5
```

## Step-07: Lihat log Autoscaler Cluster untuk memverifikasi dan memantau beban cluster ini.
```
kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler
```
- Hasil keluaran log
```log
I0607 09:14:37.793323       1 pre_filtering_processor.go:66] Skipping ip-192-168-60-30.ec2.internal - node group min size reached
I0607 09:14:37.793332       1 pre_filtering_processor.go:66] Skipping ip-192-168-27-213.ec2.internal - node group min size reached
I0607 09:14:37.793408       1 static_autoscaler.go:440] Scale down status: unneededOnly=true lastScaleUpTime=2020-06-07 09:12:27.367461648 +0000 UTC m=+37.138078060 lastScaleDownDeleteTime=2020-06-07 09:12:27.367461724 +0000 UTC m=+37.138078135 lastScaleDownFailTime=2020-06-07 09:12:27.367461801 +0000 UTC m=+37.138078213 scaleDownForbidden=false isDeleteInProgress=false scaleDownInCooldown=true
I0607 09:14:47.803891       1 static_autoscaler.go:192] Starting main loop
I0607 09:14:47.804234       1 utils.go:590] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0607 09:14:47.804251       1 filter_out_schedulable.go:65] Filtering out schedulables
I0607 09:14:47.804319       1 filter_out_schedulable.go:130] 0 other pods marked as unschedulable can be scheduled.
I0607 09:14:47.804343       1 filter_out_schedulable.go:130] 0 other pods marked as unschedulable can be scheduled.
I0607 09:14:47.804351       1 filter_out_schedulable.go:90] No schedulable pods
I0607 09:14:47.804366       1 static_autoscaler.go:334] No unschedulable pods
I0607 09:14:47.804376       1 static_autoscaler.go:381] Calculating unneeded nodes
I0607 09:14:47.804392       1 pre_filtering_processor.go:66] Skipping ip-192-168-60-30.ec2.internal - node group min size reached
I0607 09:14:47.804401       1 pre_filtering_processor.go:66] Skipping ip-192-168-27-213.ec2.internal - node group min size reached
I0607 09:14:47.804460       1 static_autoscaler.go:440] Scale down status: unneededOnly=true lastScaleUpTime=2020-06-07 09:12:27.367461648 +0000 UTC m=+37.138078060 lastScaleDownDeleteTime=2020-06-07 09:12:27.367461724 +0000 UTC m=+37.138078135 lastScaleDownFailTime=2020-06-07 09:12:27.367461801 +0000 UTC m=+37.138078213 scaleDownForbidden=false isDeleteInProgress=false scaleDownInCooldown=true

```

## Step-08: Deploy Aplikasi
```
# Deploy Aplikasi
kubectl apply -f kube-manifests/
```

## Step-09: Cluster Scale UP: Naikan skala aplikasi ke 30 pod
- Dalam 2 hingga 3 menit, satu demi satu node baru akan ditambahkan dan pod akan dijadwalkan pada mereka. 
- Jumlah maksimum node kami adalah 4 yang kami sediakan selama pembuatan nodegroup.
```
# Terminal 1: Pantau log autoscaler cluster
kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler

# Terminal 2: Tingkatkan aplikasi demo menjadi 30 pod
kubectl get pods
kubectl get nodes 
kubectl scale --replicas=30 deploy ca-demo-deployment 
kubectl get pods

# Terminal 2: Verifikasi node
kubectl get nodes -o wide
```
## Step-10: Cluster Scale DOWN: Menurunkan skala aplikasi ke 1 pod
- Mungkin perlu 10 hingga 20 menit untuk mendinginkan dan turun ke node minimum yang akan menjadi 2 yang kami konfigurasikan selama pembuatan nodegroup
```
# Terminal 1: Pantau log autoscaler cluster
kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler

# Terminal 2: Kecilkan aplikasi demo menjadi 1 pod
kubectl scale --replicas=1 deploy ca-demo-deployment 

# Terminal 2: Verifikasi node
kubectl get nodes -o wide
```
