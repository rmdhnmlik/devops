# Basic Command kubernetes

```bash
# untuk melihat node kubernetes
kubectl get nodes

# untuk melihat semua resource yang berjalan di dalam kubernetes
kubectl get all --all-namespaces

# untuk melihat namespace di kubernetes
kubectl get ns
 
```
# Basic command membuat, modifikasi, atau menghapus resource

```bash
# Apply configurasi
kubectl apply -f nama-file.yml

# Menghapus rescource
kubectl delete -f nama-file.yml atau
kubectl delete (resource) (nama)

```
# Contoh gambaran sederhana membuat resource 
buat file dengan nama pod.yml kemudian isi filenya dengan konfigurasi berikut :
```bash
apiVersion: v1
kind: Pod
metadata :
  name: nginx-demo #nama pod
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
```
konfigurasi tersebut untuk menjalankan nginx
kemudian deploy dengan command berikut :
```bash
# deploy 
kubectl apply -f pod.yml

# cek status pod apakah running atau tidak
kubectl get pod
NAME         READY   STATUS    RESTARTS   AGE
nginx-demo   1/1     Running   0          6m54s

# cek logs pada pod yang sudah kita deploy
kubectl logs -f nginx-demo

# untuk menghapus pod yang sudah kita buat 
kubectl delete -f pod.yml atau
kubectl delete pod nginx-demo

#maka akan muncl seperti berikut 
pod "nginx-demo" deleted


```
