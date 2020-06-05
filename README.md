# cka
K8S YAML templates and kubectl cheat sheet used for my CKA preparation
# Dry-run Cheat Sheet
## Pod
```terminal
kubectl run nginx --image=nginx --dry-run=client -o yaml
```
## Deployment
```terminal
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml
```
## Service
Change --type to ClusterIP/NodePort/LoadBalancer or "expose pod"/"expose replicaset":
```
kubectl expose deployment nginx --name=nginx-service --port=8080 --target-port=80 --type=NodePort --dry-run=client -o yaml
```
## ConfigMap
literals:
```terminal
kubectl create configmap myconfigmap --from-literal=my_key1=myvalue1 --from-literal=my_key2=my-value2 --dry-run=client -o yaml
```
file:
```terminal
kubectl create configmap myconfigmap --from-file=myfile.txt --dry-run=client -o yaml
```
## Secrets
encoding/decoding base64 secret (YAML should contain encoded secret):
```
root@debian:~/cka# echo -n 'mysql' | base64
bXlzcWw=
root@debian:~/cka# echo -n bXlzcWw= | base64 --decode
mysql
```
from literals:
```
kubectl create secret generic mysecret --from-literal=username1=rodrigo --from-literal=password1=mypassword -o yaml
```
from file:
```
kubectl create secret generic mysecret --from-file=myfile.txt --dry-run=client -o yaml
```
# Deployment Rollout
```
kubectl rollout history deployment
```
```

```
