# Basic Command kubernetes

```bash
# untuk melihat node kubernetes
kubectl get nodes

# untuk melihat semua resource yang berjalan di dalam kubernetes
kubectl get all --all-namespaces

# untuk melihat namespace di kubernetes
kubectl get ns
 
```
# Contoh membuat, modifikasi, atau menghapus resource

```bash
# Apply configurasi
kubectl apply -f nama-file.yml

# Menghapus rescource
kubectl delete -f nama-file.yml atau
kubectl delete (resource) (nama)

```
