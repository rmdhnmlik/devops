# 🚀 Panduan Belajar Kubernetes: Dari Dasar Hingga Production-Ready

> **Dokumentasi ini disusun secara berurutan. Pelajari dari atas ke bawah, jangan meloncat bab.**  
> Setiap bab memiliki penjelasan konsep, perintah praktik, dan checklist untuk memastikan kamu paham sebelum lanjut.

---

## 📋 Daftar Isi

1. [Pengantar & Konsep Dasar](#1-pengantar--konsep-dasar)
2. [Instalasi & Setup Environment](#2-instalasi--setup-environment)
3. [Arsitektur Kubernetes](#3-arsitektur-kubernetes)
4. [Pod — Unit Terkecil](#4-pod--unit-terkecil)
5. [ReplicaSet & Deployment](#5-replicaset--deployment)
6. [Service — Networking Dasar](#6-service--networking-dasar)
7. [ConfigMap & Secret](#7-configmap--secret)
8. [Volume & Persistent Storage](#8-volume--persistent-storage)
9. [Namespace](#9-namespace)
10. [Ingress & Load Balancer](#10-ingress--load-balancer)
11. [Resource Requests & Limits](#11-resource-requests--limits)
12. [Health Check: Liveness & Readiness Probe](#12-health-check-liveness--readiness-probe)
13. [StatefulSet](#13-statefulset)
14. [DaemonSet](#14-daemonset)
15. [Job & CronJob](#15-job--cronjob)
16. [RBAC — Keamanan & Akses](#16-rbac--keamanan--akses)
17. [Helm — Package Manager Kubernetes](#17-helm--package-manager-kubernetes)
18. [Monitoring: Prometheus & Grafana](#18-monitoring-prometheus--grafana)
19. [Logging: Fluentd / Loki](#19-logging-fluentd--loki)
20. [CI/CD dengan Kubernetes](#20-cicd-dengan-kubernetes)
21. [Production Checklist & Best Practices](#21-production-checklist--best-practices)
22. [Troubleshooting Umum](#22-troubleshooting-umum)

---

## 1. Pengantar & Konsep Dasar

### Apa itu Kubernetes?

Kubernetes (disingkat **K8s**) adalah platform open-source untuk **mengotomatisasi deployment, scaling, dan manajemen aplikasi container**.

Bayangkan kamu punya 100 container Docker yang berjalan. Siapa yang:
- Memastikan container tetap hidup jika crash?
- Mendistribusikan traffic ke container yang sehat?
- Menambah container saat traffic melonjak?
- Me-rolling update tanpa downtime?

**Jawaban: Kubernetes.**

### Mengapa Kubernetes?

| Tanpa Kubernetes | Dengan Kubernetes |
|---|---|
| Manual restart container | Auto-healing |
| Manual scaling | Auto-scaling |
| Downtime saat update | Rolling update / zero downtime |
| Konfigurasi per server | Deklaratif, satu tempat |
| Sulit monitoring | Terintegrasi dengan tools monitoring |

### Konsep Inti yang Harus Dipahami

- **Container**: Paket aplikasi + dependensi (Docker image)
- **Pod**: Satu atau lebih container yang berjalan bersama
- **Node**: Server fisik/VM tempat Pod berjalan
- **Cluster**: Kumpulan Node yang dikelola Kubernetes
- **Control Plane**: "Otak" cluster, yang mengatur segalanya
- **Manifest / YAML**: File konfigurasi deklaratif Kubernetes

### Filosofi Deklaratif

Kubernetes bekerja secara **deklaratif**: kamu mendeskripsikan *keadaan yang diinginkan*, Kubernetes yang memastikan keadaan itu tercapai.

```yaml
# Saya ingin 3 replica aplikasi web saya berjalan
replicas: 3
```

Jika satu pod mati → Kubernetes langsung buat yang baru. Kamu tidak perlu skrip restart manual.

### ✅ Checklist Bab 1
- [ ] Saya paham apa itu Kubernetes dan mengapa dibutuhkan
- [ ] Saya paham perbedaan Container vs Pod vs Node vs Cluster
- [ ] Saya paham konsep deklaratif Kubernetes

---

## 2. Instalasi & Setup Environment

### Opsi Environment Belajar

| Tool | Cocok Untuk | Kelebihan |
|---|---|---|
| **Minikube** | Belajar lokal | Mudah, ringan |
| **Kind** | Testing lokal | Cepat, Docker-based |
| **k3s** | VPS/server ringan | Minimal, production-like |
| **kubeadm** | Production | Standar resmi |
| **Rancher Desktop** | Mac/Windows | GUI + kubectl |

> 💡 **Rekomendasi belajar**: Gunakan **Minikube** dulu. Setelah paham konsep, lanjut ke kubeadm untuk cluster nyata.

### 2.1 Install kubectl (Wajib di Semua Opsi)

```bash
# Linux (Ubuntu/Debian)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verifikasi
kubectl version --client
```

### 2.2 Install Minikube (Untuk Belajar Lokal)

```bash
# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster
minikube start --cpus=2 --memory=4096

# Cek status
minikube status

# Dashboard (opsional, buka di browser)
minikube dashboard
```

### 2.3 Setup Cluster Production dengan kubeadm (3 Node)

> Asumsikan: 1 Master Node + 2 Worker Node, OS Ubuntu 22.04

#### Lakukan di SEMUA NODE:

```bash
# 1. Nonaktifkan swap (wajib!)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 2. Load kernel module
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 3. Konfigurasi sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# 4. Install containerd
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# 5. Install kubeadm, kubelet, kubectl
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

#### Hanya di MASTER NODE:

```bash
# Initialize cluster
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# Setup kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install CNI (Calico)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# Tampilkan join command untuk worker
kubeadm token create --print-join-command
```

#### Hanya di WORKER NODE:

```bash
# Jalankan output dari perintah "kubeadm token create" di atas
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

#### Verifikasi Cluster:

```bash
# Di master node
kubectl get nodes
# Output: semua node STATUS = Ready

kubectl get pods -A
# Output: semua pod di namespace kube-system harus Running
```

### ✅ Checklist Bab 2
- [ ] kubectl terinstall dan bisa dijalankan
- [ ] Cluster berjalan (minikube status / kubectl get nodes)
- [ ] Semua node STATUS = Ready

---

## 3. Arsitektur Kubernetes

### Gambaran Besar

```
┌─────────────────────────────────────────────┐
│              KUBERNETES CLUSTER              │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │          CONTROL PLANE (Master)      │   │
│  │                                      │   │
│  │  ┌──────────┐  ┌──────────────────┐  │   │
│  │  │  API     │  │   etcd           │  │   │
│  │  │  Server  │  │   (database)     │  │   │
│  │  └──────────┘  └──────────────────┘  │   │
│  │                                      │   │
│  │  ┌──────────┐  ┌──────────────────┐  │   │
│  │  │Scheduler │  │Controller Manager│  │   │
│  │  └──────────┘  └──────────────────┘  │   │
│  └──────────────────────────────────────┘   │
│                                             │
│  ┌──────────────┐  ┌──────────────────────┐ │
│  │  WORKER NODE │  │    WORKER NODE       │ │
│  │              │  │                      │ │
│  │  ┌────────┐  │  │  ┌────────────────┐  │ │
│  │  │kubelet │  │  │  │    kubelet     │  │ │
│  │  └────────┘  │  │  └────────────────┘  │ │
│  │  ┌────────┐  │  │  ┌────────────────┐  │ │
│  │  │  Pod   │  │  │  │      Pod       │  │ │
│  │  │ ┌────┐ │  │  │  │ ┌────┐ ┌────┐  │  │ │
│  │  │ │ C1 │ │  │  │  │ │ C1 │ │ C2 │  │  │ │
│  │  │ └────┘ │  │  │  │ └────┘ └────┘  │  │ │
│  │  └────────┘  │  │  └────────────────┘  │ │
│  └──────────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────┘
```

### Komponen Control Plane

| Komponen | Fungsi |
|---|---|
| **kube-apiserver** | Pintu masuk semua request. Semua komponen berkomunikasi lewat API Server |
| **etcd** | Database key-value, menyimpan semua state cluster |
| **kube-scheduler** | Memilih Node mana yang akan menjalankan Pod baru |
| **kube-controller-manager** | Menjalankan loop kontrol (pastikan desired state = actual state) |

### Komponen Worker Node

| Komponen | Fungsi |
|---|---|
| **kubelet** | Agent di setiap node, memastikan container berjalan sesuai spec |
| **kube-proxy** | Mengatur networking dan load balancing antar Pod |
| **Container Runtime** | Yang menjalankan container (containerd, CRI-O) |

### Alur Kerja: Membuat Pod Baru

```
User → kubectl apply -f pod.yaml
          ↓
      API Server (validasi, simpan ke etcd)
          ↓
      Scheduler (pilih node yang tepat)
          ↓
      kubelet di node terpilih (baca spec)
          ↓
      Container Runtime (pull image, start container)
          ↓
      Pod Running ✓
```

### ✅ Checklist Bab 3
- [ ] Saya paham perbedaan Control Plane vs Worker Node
- [ ] Saya paham fungsi masing-masing komponen
- [ ] Saya paham alur kerja dari `kubectl apply` sampai Pod berjalan

---

## 4. Pod — Unit Terkecil

### Apa itu Pod?

Pod adalah **unit deployment terkecil** di Kubernetes. Satu Pod berisi:
- Satu atau lebih container
- Storage bersama (volume)
- IP address unik dalam cluster
- Konfigurasi bersama (env var, dll)

> 💡 Dalam praktik sehari-hari, **1 Pod = 1 Container** (kecuali pattern khusus seperti sidecar).

### Manifest Pod Pertama

Buat file `pod-nginx.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    env: belajar
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### Perintah Dasar Pod

```bash
# Buat pod dari file
kubectl apply -f pod-nginx.yaml

# Lihat semua pod
kubectl get pods

# Lihat pod dengan detail lebih
kubectl get pods -o wide

# Describe pod (debug info lengkap)
kubectl describe pod nginx-pod

# Lihat log pod
kubectl logs nginx-pod

# Lihat log secara real-time (follow)
kubectl logs -f nginx-pod

# Masuk ke dalam container (exec)
kubectl exec -it nginx-pod -- /bin/bash

# Hapus pod
kubectl delete pod nginx-pod

# Hapus menggunakan file manifest
kubectl delete -f pod-nginx.yaml
```

### Port Forwarding (Akses Pod dari Lokal)

```bash
# Forward port lokal 8080 ke port 80 di pod
kubectl port-forward pod/nginx-pod 8080:80

# Buka di browser: http://localhost:8080
```

### Multi-Container Pod (Sidecar Pattern)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
  # Container utama
  - name: main-app
    image: nginx:1.25
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx

  # Sidecar: container helper
  - name: log-shipper
    image: busybox
    command: ["sh", "-c", "tail -f /logs/access.log"]
    volumeMounts:
    - name: shared-logs
      mountPath: /logs

  volumes:
  - name: shared-logs
    emptyDir: {}
```

### Lifecycle Pod

```
Pending → Running → Succeeded / Failed / CrashLoopBackOff
```

| Status | Arti |
|---|---|
| `Pending` | Pod dijadwalkan, container belum start |
| `Running` | Container berjalan normal |
| `Succeeded` | Container selesai dengan exit code 0 |
| `Failed` | Container selesai dengan exit code bukan 0 |
| `CrashLoopBackOff` | Container terus crash, Kubernetes mencoba restart |

### ✅ Checklist Bab 4
- [ ] Saya bisa membuat Pod dari file YAML
- [ ] Saya bisa melihat log pod dan masuk ke dalam container
- [ ] Saya paham status Pod dan arti CrashLoopBackOff

---

## 5. ReplicaSet & Deployment

### Mengapa Tidak Pakai Pod Langsung?

Pod bersifat **ephemeral** (sementara). Jika node mati, Pod tidak otomatis pindah. Inilah mengapa kita pakai **Deployment**.

### Deployment

Deployment adalah cara paling umum untuk deploy aplikasi. Deployment mengelola **ReplicaSet**, yang mengelola **Pod**.

```
Deployment
    └── ReplicaSet
            ├── Pod 1
            ├── Pod 2
            └── Pod 3
```

### Manifest Deployment

Buat file `deployment-nginx.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3                    # Jumlah pod yang diinginkan
  selector:
    matchLabels:
      app: nginx                 # Selector harus cocok dengan label pod
  strategy:
    type: RollingUpdate          # Strategi update
    rollingUpdate:
      maxSurge: 1                # Maksimal pod ekstra saat update
      maxUnavailable: 0          # Tidak boleh ada pod yang down saat update
  template:
    metadata:
      labels:
        app: nginx               # Label pod (harus cocok dengan selector)
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

### Perintah Deployment

```bash
# Deploy
kubectl apply -f deployment-nginx.yaml

# Lihat deployment
kubectl get deployments

# Lihat ReplicaSet
kubectl get replicasets

# Lihat pod yang dibuat deployment
kubectl get pods -l app=nginx

# Scale up/down
kubectl scale deployment nginx-deployment --replicas=5

# Update image (rolling update)
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

# Lihat status rolling update
kubectl rollout status deployment/nginx-deployment

# Lihat history rollout
kubectl rollout history deployment/nginx-deployment

# Rollback ke versi sebelumnya
kubectl rollout undo deployment/nginx-deployment

# Rollback ke versi spesifik
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

### Rolling Update vs Recreate

```yaml
# Rolling Update: pod lama dihapus satu per satu, pod baru dibuat
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0

# Recreate: semua pod dihapus dulu, baru dibuat baru (ada downtime!)
strategy:
  type: Recreate
```

### ✅ Checklist Bab 5
- [ ] Saya bisa membuat Deployment
- [ ] Saya bisa scale Deployment
- [ ] Saya bisa melakukan rolling update dan rollback
- [ ] Saya paham perbedaan RollingUpdate vs Recreate

---

## 6. Service — Networking Dasar

### Masalah Tanpa Service

Pod memiliki IP yang **berubah setiap kali** dibuat ulang. Bagaimana aplikasi lain tahu ke mana harus mengirim request?

**Solusi: Service** — memberikan IP dan DNS yang stabil untuk mengakses sekumpulan Pod.

### Tipe Service

| Tipe | Kapan Dipakai |
|---|---|
| `ClusterIP` | Akses internal dalam cluster (default) |
| `NodePort` | Expose ke luar cluster lewat port di setiap node |
| `LoadBalancer` | Butuh external load balancer (cloud provider) |
| `ExternalName` | Mapping ke nama DNS eksternal |

### ClusterIP (Internal)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP          # Default, tidak perlu ditulis jika ClusterIP
  selector:
    app: nginx             # Pilih pod dengan label ini
  ports:
  - protocol: TCP
    port: 80               # Port yang diakses dari dalam cluster
    targetPort: 80         # Port di dalam container
```

### NodePort (Akses dari Luar)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080        # Port di node (range: 30000-32767)
```

Akses: `http://<NODE_IP>:30080`

### LoadBalancer (Cloud Provider)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

### Perintah Service

```bash
# Buat service
kubectl apply -f service.yaml

# Lihat service
kubectl get services
kubectl get svc               # singkatan

# Describe service
kubectl describe svc nginx-service

# Test koneksi dari dalam cluster
kubectl run test-pod --image=busybox --rm -it -- sh
# Di dalam pod:
wget -O- http://nginx-service:80

# Hapus service
kubectl delete svc nginx-service
```

### DNS Internal Kubernetes

Setiap Service punya DNS dengan format:
```
<service-name>.<namespace>.svc.cluster.local

# Contoh:
nginx-service.default.svc.cluster.local
```

Dari Pod dalam namespace yang sama, cukup pakai nama service:
```
http://nginx-service
```

### ✅ Checklist Bab 6
- [ ] Saya bisa membuat Service ClusterIP, NodePort, LoadBalancer
- [ ] Saya paham cara kerja DNS internal Kubernetes
- [ ] Saya bisa mengakses aplikasi dari luar cluster via NodePort

---

## 7. ConfigMap & Secret

### ConfigMap

ConfigMap menyimpan **konfigurasi non-sensitif** (seperti URL database, environment, config file).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  DATABASE_HOST: "mysql-service"
  DATABASE_PORT: "3306"
  app.properties: |
    server.port=8080
    logging.level=INFO
```

### Secret

Secret menyimpan **data sensitif** (password, API key, certificate). Data di-encode base64.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  # Nilai harus di-encode base64
  # echo -n 'password123' | base64
  DATABASE_PASSWORD: cGFzc3dvcmQxMjM=
  API_KEY: bXlzZWNyZXRrZXk=
```

> ⚠️ **Perhatian**: Secret base64 bukan enkripsi! Untuk production, gunakan **Sealed Secrets** atau **HashiCorp Vault**.

```bash
# Cara encode base64
echo -n 'password123' | base64
# Output: cGFzc3dvcmQxMjM=

# Cara decode
echo 'cGFzc3dvcmQxMjM=' | base64 --decode
```

### Cara Menggunakan di Pod

#### Sebagai Environment Variable

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    # Dari ConfigMap
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV

    # Dari Secret
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: DATABASE_PASSWORD

    # Semua key dari ConfigMap sekaligus
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secret
```

#### Sebagai Volume (File)

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config

  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

### Perintah ConfigMap & Secret

```bash
# Buat dari file
kubectl apply -f configmap.yaml

# Buat dari command line
kubectl create configmap my-config --from-literal=KEY=value
kubectl create secret generic my-secret --from-literal=PASSWORD=rahasia

# Lihat
kubectl get configmaps
kubectl get secrets

# Decode secret
kubectl get secret app-secret -o jsonpath='{.data.DATABASE_PASSWORD}' | base64 --decode
```

### ✅ Checklist Bab 7
- [ ] Saya bisa membuat ConfigMap dan Secret
- [ ] Saya bisa meng-inject ConfigMap/Secret sebagai env var
- [ ] Saya bisa meng-inject ConfigMap/Secret sebagai volume
- [ ] Saya paham batasan keamanan Secret di Kubernetes

---

## 8. Volume & Persistent Storage

### Masalah: Data Container Hilang

Ketika container restart, semua data di dalamnya **hilang**. Kita butuh storage yang persisten.

### Jenis Volume

| Tipe | Deskripsi | Kapan Dipakai |
|---|---|---|
| `emptyDir` | Temporary, hilang saat pod mati | Sharing data antar container dalam pod |
| `hostPath` | Mount folder dari node | Debug, DaemonSet |
| `PersistentVolume` | Storage persisten, berbagai backend | Database, aplikasi stateful |
| `configMap` / `secret` | Inject konfigurasi | Konfigurasi aplikasi |

### emptyDir (Sharing Antar Container)

```yaml
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: shared
      mountPath: /data

  - name: sidecar
    image: busybox
    volumeMounts:
    - name: shared
      mountPath: /data

  volumes:
  - name: shared
    emptyDir: {}
```

### PersistentVolume (PV) & PersistentVolumeClaim (PVC)

**Alur kerja storage persisten:**

```
Admin buat PV (storage pool)
  ↓
Developer buat PVC (request storage)
  ↓
Kubernetes bind PVC ke PV
  ↓
Pod gunakan PVC
```

#### PersistentVolume (dibuat admin)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce         # RWO: 1 node, RWX: banyak node, ROX: read-only banyak node
  persistentVolumeReclaimPolicy: Retain   # Retain / Delete / Recycle
  storageClassName: standard
  hostPath:                 # Untuk development lokal
    path: /data/pv
```

#### PersistentVolumeClaim (dibuat developer)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

#### Pakai PVC di Pod

```yaml
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    volumeMounts:
    - name: mysql-storage
      mountPath: /var/lib/mysql

  volumes:
  - name: mysql-storage
    persistentVolumeClaim:
      claimName: pvc-data
```

### StorageClass (Dynamic Provisioning)

Di cloud (AWS, GCP, Azure), storage dibuat otomatis tanpa perlu buat PV manual:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs   # atau gce-pd, azure-disk
parameters:
  type: gp3
reclaimPolicy: Delete
allowVolumeExpansion: true
```

```bash
# Lihat StorageClass yang tersedia
kubectl get storageclass

# Lihat PV dan PVC
kubectl get pv
kubectl get pvc
```

### ✅ Checklist Bab 8
- [ ] Saya paham perbedaan emptyDir vs PersistentVolume
- [ ] Saya bisa membuat PV dan PVC
- [ ] Saya bisa menghubungkan PVC ke Pod
- [ ] Saya paham konsep Dynamic Provisioning

---

## 9. Namespace

### Apa itu Namespace?

Namespace adalah **partisi virtual** di dalam cluster. Berguna untuk:
- Memisahkan environment (dev, staging, production)
- Multi-team dalam satu cluster
- Membatasi resource per namespace

### Namespace Default di Kubernetes

| Namespace | Fungsi |
|---|---|
| `default` | Namespace default jika tidak disebutkan |
| `kube-system` | Komponen sistem Kubernetes |
| `kube-public` | Resource yang bisa diakses semua user |
| `kube-node-lease` | Heartbeat node |

### Membuat & Menggunakan Namespace

```bash
# Buat namespace
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production

# Atau dengan YAML
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    env: dev
EOF

# Lihat namespace
kubectl get namespaces
kubectl get ns          # singkatan

# Deploy ke namespace tertentu
kubectl apply -f deployment.yaml -n development

# Lihat resource di namespace tertentu
kubectl get pods -n development
kubectl get all -n development

# Lihat resource di semua namespace
kubectl get pods -A
kubectl get pods --all-namespaces
```

### Set Default Namespace

```bash
# Ubah namespace default untuk context saat ini
kubectl config set-context --current --namespace=development

# Cek context saat ini
kubectl config current-context

# Kembali ke default
kubectl config set-context --current --namespace=default
```

### Namespace dalam Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: development    # Tentukan namespace di sini
spec:
  ...
```

### Resource Quota per Namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
    pods: "20"
    services: "10"
    persistentvolumeclaims: "5"
```

```bash
kubectl describe resourcequota dev-quota -n development
```

### ✅ Checklist Bab 9
- [ ] Saya bisa membuat dan menggunakan Namespace
- [ ] Saya bisa deploy ke namespace tertentu
- [ ] Saya bisa mengatur ResourceQuota per namespace

---

## 10. Ingress & Load Balancer

### Masalah dengan NodePort / LoadBalancer

- **NodePort**: Port acak (30000+), tidak user-friendly
- **LoadBalancer**: Setiap service butuh 1 IP publik (mahal di cloud!)

**Solusi: Ingress** — satu entry point untuk semua service, routing berdasarkan path/domain.

### Ingress Architecture

```
Internet
    ↓
Ingress Controller (nginx / traefik)
    ↓
┌──────────────────────────────┐
│         Ingress Rules        │
│ example.com/api  → api-svc   │
│ example.com/web  → web-svc   │
│ app.example.com  → app-svc   │
└──────────────────────────────┘
    ↓
Services & Pods
```

### Install Ingress Controller (NGINX)

```bash
# Untuk cluster biasa
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/cloud/deploy.yaml

# Untuk Minikube
minikube addons enable ingress

# Verifikasi
kubectl get pods -n ingress-nginx
```

### Manifest Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
  # Routing berdasarkan path
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80

  # Routing berdasarkan subdomain
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80

  # TLS
  tls:
  - hosts:
    - example.com
    - admin.example.com
    secretName: tls-secret    # Secret berisi sertifikat
```

### TLS dengan cert-manager (Auto SSL)

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.yaml

# Buat ClusterIssuer (Let's Encrypt)
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

```yaml
# Di Ingress, tambahkan annotation:
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

### ✅ Checklist Bab 10
- [ ] Ingress Controller terinstall
- [ ] Saya bisa membuat Ingress dengan routing path
- [ ] Saya bisa membuat Ingress dengan routing subdomain
- [ ] Saya paham cara setup TLS

---

## 11. Resource Requests & Limits

### Konsep Penting

| | Requests | Limits |
|---|---|---|
| **Fungsi** | Minimum yang dijamin | Maksimum yang diizinkan |
| **Scheduling** | Scheduler pakai ini | Tidak dipakai scheduling |
| **CPU** | Dipertahankan | Throttling jika melebihi |
| **Memory** | Dipertahankan | OOMKilled jika melebihi |

### Satuan Resource

```
# CPU
1 CPU = 1000m (millicore)
0.5 CPU = 500m
100m = 0.1 CPU (10% dari 1 core)

# Memory
1Gi = 1024 MiB
1Mi = 1024 KiB
```

### Contoh Penggunaan

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    resources:
      requests:
        cpu: "100m"       # Butuh minimal 0.1 CPU
        memory: "128Mi"   # Butuh minimal 128 MB RAM
      limits:
        cpu: "500m"       # Maksimal 0.5 CPU
        memory: "256Mi"   # Maksimal 256 MB RAM
```

### LimitRange (Default Limits per Namespace)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: development
spec:
  limits:
  - type: Container
    default:             # Limit default jika tidak ditentukan
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:      # Request default
      cpu: "100m"
      memory: "128Mi"
    max:                 # Batas maksimum yang boleh di-set
      cpu: "2"
      memory: "1Gi"
    min:                 # Batas minimum
      cpu: "50m"
      memory: "64Mi"
```

### Horizontal Pod Autoscaler (HPA)

HPA otomatis scale Pod berdasarkan CPU/Memory usage:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # Scale up jika CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
# Lihat HPA
kubectl get hpa

# Monitor HPA
kubectl describe hpa nginx-hpa
```

> ⚠️ HPA membutuhkan **metrics-server** terinstall di cluster.

```bash
# Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Cek resource usage
kubectl top pods
kubectl top nodes
```

### ✅ Checklist Bab 11
- [ ] Semua deployment saya sudah punya requests dan limits
- [ ] Saya bisa membuat LimitRange
- [ ] Saya bisa membuat HPA dan memahami cara kerjanya

---

## 12. Health Check: Liveness & Readiness Probe

### Mengapa Perlu Health Check?

- **Liveness Probe**: "Apakah container masih hidup?" — Jika gagal, container **di-restart**
- **Readiness Probe**: "Apakah container siap menerima traffic?" — Jika gagal, pod **dikeluarkan dari load balancer**
- **Startup Probe**: "Apakah aplikasi sudah selesai startup?" — Mencegah liveness probe terlalu cepat kill container

### Tipe Probe

| Tipe | Cara Kerja |
|---|---|
| `httpGet` | HTTP GET ke endpoint, sukses jika 200-399 |
| `tcpSocket` | Coba koneksi TCP ke port |
| `exec` | Jalankan command, sukses jika exit code 0 |
| `grpc` | gRPC health check |

### Contoh Lengkap

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8080

    # Startup Probe: tunggu hingga app selesai start
    startupProbe:
      httpGet:
        path: /health
        port: 8080
      failureThreshold: 30      # Coba maksimal 30 kali
      periodSeconds: 10         # Setiap 10 detik
      # Total wait time: 30 x 10 = 300 detik

    # Liveness Probe: restart jika tidak sehat
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      initialDelaySeconds: 30   # Tunggu 30 detik sebelum mulai cek
      periodSeconds: 10         # Cek setiap 10 detik
      timeoutSeconds: 5         # Timeout per cek
      failureThreshold: 3       # Gagal 3 kali = restart

    # Readiness Probe: keluarkan dari LB jika belum siap
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      successThreshold: 1       # Sukses 1 kali = siap
      failureThreshold: 3
```

### Contoh exec Probe

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Contoh TCP Probe

```yaml
livenessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 15
  periodSeconds: 20
```

### ✅ Checklist Bab 12
- [ ] Semua deployment production saya sudah punya liveness dan readiness probe
- [ ] Saya paham perbedaan liveness vs readiness vs startup probe
- [ ] Saya tidak menggunakan initialDelaySeconds terlalu kecil

---

## 13. StatefulSet

### Kapan Pakai StatefulSet?

Untuk aplikasi **stateful** seperti database, message queue yang membutuhkan:
- Identity Pod yang **stabil dan persisten** (nama tidak berubah)
- **Ordered deployment** (pod-0 dulu, baru pod-1, dst)
- **Storage persisten per pod** (setiap pod punya PVC sendiri)

### Perbedaan Deployment vs StatefulSet

| | Deployment | StatefulSet |
|---|---|---|
| Nama Pod | Random: `app-abc12` | Ordered: `app-0`, `app-1` |
| Scaling | Semua sekaligus | Satu per satu |
| Storage | Shared PVC | PVC per pod |
| Cocok untuk | Stateless app | Database, Kafka, Redis cluster |

### Manifest StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"      # Headless service (wajib!)
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql

  # PVC dibuat otomatis per pod
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi

---
# Headless Service (wajib untuk StatefulSet)
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None           # Headless: tidak ada load balancing
  selector:
    app: mysql
  ports:
  - port: 3306
```

### DNS di StatefulSet

Setiap Pod StatefulSet punya DNS stabil:
```
<pod-name>.<service-name>.<namespace>.svc.cluster.local

# Contoh:
mysql-0.mysql.default.svc.cluster.local
mysql-1.mysql.default.svc.cluster.local
mysql-2.mysql.default.svc.cluster.local
```

### ✅ Checklist Bab 13
- [ ] Saya paham kapan pakai StatefulSet vs Deployment
- [ ] Saya bisa membuat StatefulSet dengan PVC per pod
- [ ] Saya paham pentingnya Headless Service untuk StatefulSet

---

## 14. DaemonSet

### Apa itu DaemonSet?

DaemonSet memastikan **setiap node** menjalankan satu copy Pod. Cocok untuk:
- Log collector (Fluentd, Filebeat)
- Monitoring agent (node-exporter)
- Network plugin (CNI)
- Storage daemon

### Manifest DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      # Akses ke filesystem dan network node
      hostNetwork: true
      hostPID: true
      tolerations:                  # Jalankan di semua node termasuk master
      - operator: Exists
        effect: NoSchedule
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.7.0
        ports:
        - containerPort: 9100
        securityContext:
          privileged: true
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
```

### ✅ Checklist Bab 14
- [ ] Saya paham kapan menggunakan DaemonSet
- [ ] Saya bisa membuat DaemonSet

---

## 15. Job & CronJob

### Job

Job menjalankan Pod hingga selesai (exit code 0). Cocok untuk:
- Batch processing
- Database migration
- One-time tasks

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  completions: 1          # Harus selesai 1 kali
  parallelism: 1          # Jalankan 1 pod sekaligus
  backoffLimit: 3         # Retry maksimal 3 kali
  activeDeadlineSeconds: 300  # Timeout 5 menit
  template:
    spec:
      restartPolicy: OnFailure  # Never / OnFailure (wajib diset)
      containers:
      - name: migration
        image: myapp:1.0
        command: ["python", "manage.py", "migrate"]
        env:
        - name: DB_HOST
          value: mysql-service
```

```bash
# Lihat job
kubectl get jobs

# Lihat pod hasil job
kubectl get pods --selector=job-name=db-migration

# Lihat log job
kubectl logs job/db-migration
```

### CronJob

CronJob menjalankan Job secara terjadwal (seperti cron di Linux).

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-database
spec:
  schedule: "0 2 * * *"    # Setiap jam 2 pagi (format cron)
  concurrencyPolicy: Forbid # Jangan jalankan jika yang sebelumnya masih berjalan
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: mysql:8.0
            command:
            - /bin/sh
            - -c
            - "mysqldump -h mysql-service -u root -p$MYSQL_PASSWORD mydb > /backup/db-$(date +%Y%m%d).sql"
            env:
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
```

**Format Cron:**
```
# ┌──── Menit (0-59)
# │ ┌──── Jam (0-23)
# │ │ ┌──── Hari (1-31)
# │ │ │ ┌──── Bulan (1-12)
# │ │ │ │ ┌──── Hari Minggu (0-7, 0 dan 7 = Minggu)
# │ │ │ │ │
  * * * * *

0 2 * * *     = Setiap hari jam 02:00
*/15 * * * *  = Setiap 15 menit
0 0 * * 0     = Setiap Minggu jam 00:00
```

### ✅ Checklist Bab 15
- [ ] Saya bisa membuat Job dan memantau hasilnya
- [ ] Saya bisa membuat CronJob dengan jadwal yang benar
- [ ] Saya paham perbedaan restartPolicy: Never vs OnFailure

---

## 16. RBAC — Keamanan & Akses

### Konsep RBAC

**Role-Based Access Control** mengontrol siapa bisa melakukan apa terhadap resource apa.

```
ServiceAccount / User / Group
    ↓ (dikoneksikan via RoleBinding)
Role / ClusterRole
    ↓ (mendefinisikan)
Permissions (verbs: get, list, create, delete, ...)
    ↓ (pada)
Resources (pods, deployments, secrets, ...)
```

### Komponen RBAC

| Komponen | Scope | Keterangan |
|---|---|---|
| `Role` | 1 Namespace | Hak akses dalam satu namespace |
| `ClusterRole` | Seluruh Cluster | Hak akses global |
| `RoleBinding` | 1 Namespace | Bind Role ke User/SA |
| `ClusterRoleBinding` | Seluruh Cluster | Bind ClusterRole ke User/SA |

### ServiceAccount

```yaml
# Buat ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
```

### Role (Namespace-level)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]           # "" = core API group
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
```

### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-pod-reader
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: production
- kind: User            # Bisa bind ke user juga
  name: john@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole & ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin-readonly
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bind-readonly
subjects:
- kind: Group
  name: devops-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin-readonly
  apiGroup: rbac.authorization.k8s.io
```

### Cek Hak Akses

```bash
# Apakah saya bisa list pods?
kubectl auth can-i list pods

# Apakah user john bisa delete pods di namespace production?
kubectl auth can-i delete pods --namespace=production --as=john

# Lihat semua yang bisa dilakukan
kubectl auth can-i --list
```

### Network Policy (Firewall antar Pod)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend            # Berlaku untuk pod backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend       # Hanya izinkan dari pod frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database       # Hanya boleh keluar ke database
    ports:
    - protocol: TCP
      port: 3306
```

### ✅ Checklist Bab 16
- [ ] Saya bisa membuat ServiceAccount, Role, dan RoleBinding
- [ ] Saya tidak menggunakan `default` ServiceAccount di production
- [ ] Saya bisa mengecek hak akses dengan `kubectl auth can-i`
- [ ] Saya paham konsep Network Policy

---

## 17. Helm — Package Manager Kubernetes

### Apa itu Helm?

Helm adalah package manager untuk Kubernetes. Seperti `apt` untuk Ubuntu atau `pip` untuk Python.

- **Chart**: Paket Helm (kumpulan template Kubernetes)
- **Release**: Instance chart yang ter-deploy
- **Repository**: Tempat menyimpan chart

### Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verifikasi
helm version
```

### Perintah Helm Dasar

```bash
# Tambah repository
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Cari chart
helm search repo nginx
helm search hub wordpress

# Install chart
helm install my-nginx bitnami/nginx
helm install my-nginx bitnami/nginx --namespace web --create-namespace

# Install dengan custom values
helm install my-nginx bitnami/nginx -f custom-values.yaml
helm install my-nginx bitnami/nginx --set service.type=NodePort

# Lihat release
helm list
helm list -A   # semua namespace

# Upgrade release
helm upgrade my-nginx bitnami/nginx --set replicaCount=3

# Rollback
helm rollback my-nginx 1  # rollback ke revision 1

# Uninstall
helm uninstall my-nginx

# Lihat values yang tersedia
helm show values bitnami/nginx
```

### Membuat Chart Sendiri

```bash
# Buat struktur chart baru
helm create my-app

# Struktur folder:
# my-app/
# ├── Chart.yaml          # Metadata chart
# ├── values.yaml         # Default values
# ├── templates/          # Template manifest Kubernetes
# │   ├── deployment.yaml
# │   ├── service.yaml
# │   ├── ingress.yaml
# │   └── _helpers.tpl    # Template helpers
# └── charts/             # Dependencies
```

#### Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: My application Helm chart
type: application
version: 0.1.0          # Versi chart
appVersion: "1.0.0"     # Versi aplikasi
```

#### values.yaml

```yaml
replicaCount: 2

image:
  repository: myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

ingress:
  enabled: false
  host: ""
```

#### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: 80
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

```bash
# Test render template (tanpa deploy)
helm template my-app ./my-app

# Install chart lokal
helm install my-app ./my-app

# Install dengan override values
helm install my-app ./my-app --set replicaCount=3 --set image.tag=2.0.0

# Package chart
helm package ./my-app
```

### ✅ Checklist Bab 17
- [ ] Helm terinstall
- [ ] Saya bisa install chart dari repository
- [ ] Saya bisa membuat chart sederhana sendiri
- [ ] Saya bisa upgrade dan rollback Helm release

---

## 18. Monitoring: Prometheus & Grafana

### Stack Monitoring Kubernetes

```
Aplikasi / Node / K8s Components
    ↓ (expose metrics)
Prometheus (scrape & store metrics)
    ↓ (query PromQL)
Grafana (visualisasi dashboard)
    ↓ (trigger alerts)
Alertmanager (kirim notifikasi ke Slack, Email, PagerDuty)
```

### Install dengan Helm (kube-prometheus-stack)

```bash
# Tambah repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=rahasia123 \
  --set prometheus.prometheusSpec.retention=30d

# Cek pod
kubectl get pods -n monitoring

# Akses Grafana
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
# Buka: http://localhost:3000 (admin / rahasia123)

# Akses Prometheus
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
```

### Expose Metrics dari Aplikasi

Aplikasi harus expose endpoint `/metrics` dalam format Prometheus:

```python
# Contoh Python (Flask + prometheus-client)
from prometheus_client import Counter, generate_latest

requests_total = Counter('http_requests_total', 'Total requests')

@app.route('/metrics')
def metrics():
    return generate_latest()
```

### ServiceMonitor (Prometheus scrape konfigurasi)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring
  labels:
    release: monitoring       # Harus match dengan label Prometheus
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    matchNames:
    - production
  endpoints:
  - port: metrics
    path: /metrics
    interval: 30s
```

### PrometheusRule (Alert Rules)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
  - name: app.rules
    interval: 30s
    rules:
    # Alert jika pod tidak running
    - alert: PodNotRunning
      expr: kube_pod_status_phase{phase!="Running"} > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} tidak running"
        description: "Pod {{ $labels.pod }} di namespace {{ $labels.namespace }} sudah tidak running selama 5 menit"

    # Alert jika CPU usage tinggi
    - alert: HighCPUUsage
      expr: sum(rate(container_cpu_usage_seconds_total[5m])) by (pod) > 0.8
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "CPU tinggi di pod {{ $labels.pod }}"
```

### Query PromQL Berguna

```promql
# CPU usage semua pod
rate(container_cpu_usage_seconds_total[5m])

# Memory usage
container_memory_working_set_bytes

# Jumlah pod running per deployment
kube_deployment_status_replicas_ready

# HTTP request rate (jika pakai nginx ingress)
rate(nginx_ingress_controller_requests[5m])

# Error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
```

### ✅ Checklist Bab 18
- [ ] Prometheus dan Grafana terinstall
- [ ] Saya bisa mengakses Grafana dashboard
- [ ] Saya bisa membuat ServiceMonitor
- [ ] Saya bisa membuat alert rules

---

## 19. Logging: Fluentd / Loki

### Stack Logging

```
Pod Logs
    ↓ (collect)
Fluentd / Promtail (log collector, DaemonSet)
    ↓ (store)
Elasticsearch / Loki (log storage)
    ↓ (visualize)
Kibana / Grafana (tampilkan log)
```

### Opsi 1: EFK Stack (Elasticsearch + Fluentd + Kibana)

```bash
# Untuk production skala besar
helm repo add elastic https://helm.elastic.co

helm install elasticsearch elastic/elasticsearch \
  --namespace logging --create-namespace \
  --set replicas=1 \
  --set resources.requests.memory=512Mi

helm install kibana elastic/kibana --namespace logging

helm install fluentd stable/fluentd --namespace logging
```

### Opsi 2: PLG Stack (Promtail + Loki + Grafana) - Lebih Ringan

```bash
helm repo add grafana https://grafana.github.io/helm-charts

helm install loki grafana/loki-stack \
  --namespace logging --create-namespace \
  --set grafana.enabled=true \
  --set promtail.enabled=true
```

### Best Practice Logging

```yaml
# Aplikasi harus log ke stdout/stderr (bukan file)
# Kubernetes otomatis collect log dari stdout/stderr

# Format log yang direkomendasikan: JSON
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "INFO",
  "message": "Request processed",
  "request_id": "abc-123",
  "duration_ms": 45,
  "user_id": 1001
}
```

### ✅ Checklist Bab 19
- [ ] Logging stack terinstall (EFK atau PLG)
- [ ] Semua pod log ke stdout/stderr
- [ ] Saya bisa search log dari Kibana/Grafana

---

## 20. CI/CD dengan Kubernetes

### Alur CI/CD

```
Developer push code
    ↓
CI Pipeline (GitHub Actions / GitLab CI / Jenkins)
    ├── Build Docker image
    ├── Run unit tests
    ├── Push image ke registry (Docker Hub, ECR, GCR)
    └── Update Kubernetes manifest
        ↓
CD Pipeline
    └── kubectl apply / helm upgrade → Kubernetes
```

### Contoh GitHub Actions Pipeline

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy to Kubernetes

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: myapp
  REGISTRY: ghcr.io/${{ github.repository_owner }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
    - uses: actions/checkout@v4

    - name: Login to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=sha,prefix={{branch}}-

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Setup kubectl
      uses: azure/setup-kubectl@v3

    - name: Configure kubeconfig
      run: |
        mkdir -p ~/.kube
        echo "${{ secrets.KUBECONFIG }}" > ~/.kube/config

    - name: Deploy to Kubernetes
      run: |
        # Update image di deployment
        kubectl set image deployment/myapp \
          myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
          -n production

        # Tunggu rollout selesai
        kubectl rollout status deployment/myapp -n production --timeout=5m
```

### GitOps dengan ArgoCD

**GitOps**: Kubernetes manifest disimpan di Git. ArgoCD otomatis sync cluster dengan Git.

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Akses ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

```yaml
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/k8s-manifests
    targetRevision: HEAD
    path: production/my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true       # Hapus resource yang tidak ada di Git
      selfHeal: true    # Auto fix jika ada perubahan manual
    syncOptions:
    - CreateNamespace=true
```

### ✅ Checklist Bab 20
- [ ] CI pipeline bisa build dan push Docker image
- [ ] CD pipeline bisa deploy ke Kubernetes
- [ ] Saya paham konsep GitOps
- [ ] Secrets tidak disimpan di Git (pakai GitHub Secrets / Vault)

---

## 21. Production Checklist & Best Practices

### 🔒 Keamanan

```yaml
# 1. Security Context: jalankan sebagai non-root
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL

# 2. Selalu tentukan image tag spesifik (jangan :latest)
image: nginx:1.25.3   # ✅
image: nginx:latest   # ❌

# 3. Scan image untuk vulnerabilities
# Gunakan: Trivy, Snyk, atau Docker Scout
trivy image nginx:1.25.3
```

### 📊 Resource Management

```yaml
# Semua container WAJIB punya requests dan limits
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"

# Gunakan HPA untuk auto-scaling
# Gunakan ResourceQuota per namespace
# Gunakan LimitRange untuk default limits
```

### 🏥 High Availability

```yaml
# Minimal 2 replicas untuk production
replicas: 2

# PodDisruptionBudget: pastikan minimal 1 pod selalu running
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 1   # atau maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp

# Spread pods di banyak node
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: myapp
```

### 🔄 Update Strategy

```yaml
# Selalu gunakan RollingUpdate
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

### 💾 Backup & Disaster Recovery

```bash
# Backup etcd (lakukan reguler!)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Backup semua manifest
kubectl get all --all-namespaces -o yaml > cluster-backup.yaml

# Backup dengan Velero (rekomendasi untuk production)
helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  --set configuration.provider=aws \
  --set configuration.backupStorageLocation.bucket=my-backup-bucket
```

### 📝 Production Checklist Lengkap

#### Cluster

- [ ] Master node minimal 3 (untuk HA etcd)
- [ ] Worker node sesuai kebutuhan, pisahkan beban kerja
- [ ] Kubernetes versi tidak lebih dari N-2 dari stable terbaru
- [ ] etcd backup otomatis berjalan
- [ ] Node monitoring (disk, CPU, memory alerts)

#### Aplikasi

- [ ] Semua deployment punya `requests` dan `limits`
- [ ] Semua deployment punya liveness dan readiness probe
- [ ] Tidak ada container yang jalan sebagai root
- [ ] Image tag spesifik (bukan `:latest`)
- [ ] Image sudah di-scan untuk vulnerability
- [ ] `imagePullPolicy: IfNotPresent` atau `Always` (bukan `Never`)

#### Keamanan

- [ ] RBAC diaktifkan dan dikonfigurasi dengan benar
- [ ] Tidak ada ServiceAccount dengan hak berlebihan
- [ ] Secret di-encrypt at rest (gunakan KMS)
- [ ] Network Policy dikonfigurasi (deny-all by default)
- [ ] Audit logging diaktifkan
- [ ] Ingress menggunakan TLS
- [ ] Pod Security Standards/Admission Controller dikonfigurasi

#### Observability

- [ ] Prometheus + Grafana berjalan
- [ ] Alert untuk node down, pod crash, high resource usage
- [ ] Centralized logging (EFK/PLG)
- [ ] Distributed tracing (Jaeger/Zipkin) jika diperlukan
- [ ] Uptime monitoring (Blackbox Exporter)
- [ ] Dashboard SLO/SLA

#### CI/CD

- [ ] Pipeline otomatis untuk build, test, deploy
- [ ] Tidak ada credentials hardcoded di manifest
- [ ] Staging environment sebelum production
- [ ] Rollback plan tersedia
- [ ] GitOps dengan ArgoCD/Flux (rekomendasi)

---

## 22. Troubleshooting Umum

### Perintah Debugging Esensial

```bash
# Lihat event terbaru di namespace
kubectl get events --sort-by='.lastTimestamp' -n production

# Describe resource (paling informatif untuk debug)
kubectl describe pod <pod-name>
kubectl describe node <node-name>

# Log container
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>  # multi-container pod
kubectl logs <pod-name> --previous           # log container sebelumnya
kubectl logs <pod-name> --tail=100 -f       # 100 baris terakhir, follow

# Exec ke dalam container
kubectl exec -it <pod-name> -- /bin/sh

# Check resource usage
kubectl top pods -n production
kubectl top nodes
```

### Masalah Umum dan Solusi

#### Pod Status: `Pending`

```bash
kubectl describe pod <pod-name>
# Perhatikan bagian "Events:" di bawah

# Kemungkinan penyebab:
# 1. Tidak ada node yang cukup resource
kubectl top nodes
kubectl describe nodes

# 2. PVC tidak ter-bound
kubectl get pvc

# 3. Image pull error
# Cek Events: "Failed to pull image"
kubectl describe pod <pod>
```

#### Pod Status: `CrashLoopBackOff`

```bash
# Lihat log container yang crash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # log dari run sebelumnya

# Kemungkinan:
# - Aplikasi error saat startup
# - Environment variable / config salah
# - Port sudah dipakai
# - Permission denied
```

#### Pod Status: `ImagePullBackOff`

```bash
kubectl describe pod <pod-name>
# Cek Events untuk detail error

# Kemungkinan:
# 1. Nama image salah atau tidak ada
# 2. Image bersifat private, perlu ImagePullSecret

# Solusi: tambahkan imagePullSecret
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=password

# Di pod spec:
spec:
  imagePullSecrets:
  - name: regcred
```

#### Service Tidak Bisa Diakses

```bash
# 1. Pastikan selector cocok dengan label pod
kubectl get pods --show-labels
kubectl describe svc <service-name>

# 2. Test koneksi dari dalam cluster
kubectl run debug --image=busybox --rm -it -- sh
wget -O- http://<service-name>:<port>

# 3. Cek endpoint
kubectl get endpoints <service-name>
# Jika ENDPOINTS = <none>, selector tidak match
```

#### Node NotReady

```bash
kubectl describe node <node-name>
# Perhatikan section Conditions dan Events

# SSH ke node, cek kubelet
systemctl status kubelet
journalctl -u kubelet -n 100

# Kemungkinan:
# - Disk penuh (cek /var/lib/kubelet)
# - Memory habis
# - kubelet tidak bisa reach API server
```

#### Deployment Stuck / Tidak Mau Rollout

```bash
kubectl rollout status deployment/<name>
kubectl describe deployment/<name>

# Cek ReplicaSet
kubectl get rs
kubectl describe rs <rs-name>

# Force restart deployment (buat pod baru tanpa ubah spec)
kubectl rollout restart deployment/<name>
```

#### etcd Full / Cluster Tidak Stabil

```bash
# Cek ukuran etcd
ETCDCTL_API=3 etcdctl endpoint status \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --write-out=table

# Compact dan defrag etcd
ETCDCTL_API=3 etcdctl defrag --endpoints=https://127.0.0.1:2379 \
  --cacert=... --cert=... --key=...
```

### Tips Debug Cepat

```bash
# Buat pod debug dengan banyak tools
kubectl run debug --image=nicolaka/netshoot --rm -it -- /bin/bash

# Di dalam pod, bisa pakai: curl, dig, nslookup, ping, netstat, tcpdump

# Test DNS resolution
nslookup my-service.my-namespace.svc.cluster.local

# Test koneksi TCP
nc -zv my-service 80

# Lihat network connections
ss -tlnp
```

---

## 📚 Referensi & Sumber Belajar Lanjutan

| Sumber | Link | Keterangan |
|---|---|---|
| Dokumentasi Resmi | kubernetes.io/docs | Paling akurat dan lengkap |
| Kubernetes The Hard Way | github.com/kelseyhightower/kubernetes-the-hard-way | Pelajari dari scratch |
| kubectl Cheat Sheet | kubernetes.io/docs/reference/kubectl/cheatsheet | Referensi perintah |
| CKAD Exam Guide | training.linuxfoundation.org/certification/ckad | Sertifikasi developer |
| CKA Exam Guide | training.linuxfoundation.org/certification/cka | Sertifikasi admin |
| Play with Kubernetes | labs.play-with-k8s.com | Lab online gratis |
| Killercoda | killercoda.com/kubernetes | Latihan interaktif |

---

## 🎯 Roadmap Belajar yang Disarankan

```
Week 1-2:  Bab 1-5   → Dasar K8s, Pod, Deployment
Week 3:    Bab 6-9   → Networking, Config, Storage, Namespace
Week 4:    Bab 10-12 → Ingress, Resource Management, Health Check
Week 5:    Bab 13-16 → StatefulSet, DaemonSet, Job, RBAC
Week 6:    Bab 17-18 → Helm, Monitoring
Week 7-8:  Bab 19-21 → Logging, CI/CD, Production Best Practices
Ongoing:   Bab 22    → Troubleshooting (gunakan saat ada masalah)
```

> 💪 **Tips**: Praktikkan setiap bab di Minikube sebelum lanjut. Tidak ada cara belajar Kubernetes yang lebih efektif daripada langsung mencoba.

---

*Dokumen ini dibuat untuk belajar Kubernetes secara sistematis. Update sesuai perkembangan versi Kubernetes terbaru.*
