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
Create deployment with YAML file and --record to track version changes:
```
kubectl create -f <deployment file name>.yaml --record
```
Check Deployment history:
```
kubectl rollout history deployment <deployment name>
```
Update deployment image:
```
kubectl set image deployment nginx nginx=nginx:1.19
```
Confirm rollout was successful:
```
kubectl rollout status deployment <deployment name>
```
Rollback to previous version:
```
kubectl rollout undo deployment <deployment name>
```
Alternatively, to specific revision number:
```
kubectl rollout undo deployment nginx --to-revision=<revision number>
```
# Node Maintenance
Draining node so pods are gracefully terminated and re-created on another node and node tainted to be unscheduleable:
```
kubectl drain <node_name> --ignore-daemonsets
```
Uncordoning node (pods don't fall back automatically):
```
kubectl uncordon <node_name>
```
Cordoning (makes node unscheduleable but don't move pods to other nodes):
```
kubectl cordon <node_name>
```

# kubeadm Upgrade Process
## Master node
Check which versions are available for upgrade:
```
kubeadm upgrade plan
```
Upgrade kubeadm:
```
apt-get upgrade -y kubeadm=1.12.0-00
```
Perform master node's components upgrade (api-server, controller-manager and kube-proxy):
```
kubeadm upgrade apply v1.12.0
```
kubectl get nodes still show old version because kubelet has not been upgraded yet
Upgrade kubelet:
```
apt-get upgrade -y kubelet=1.12.0-00
```
Restart kubelet
```
systemctl restart kubelet
```
## Worker nodes
Drain worker node:
```
kubectl drain <node>
```
Upgrade kubeadm:
```
apt-get upgrade -y kubeadm=10.12.0-00
```
Upgrade kubelet daemon:
```
apt-get upgrade -y kubelet=1.12.0-00
```
Update node config for new kubelet version:
```
kubeadm upgrade node config --kubelet-version v1.12.0
```
Restart kubelet daemon:
```
systemctl restart kubelet
```
Uncordon node:
```
kubectl uncordon <node>
```
Repeat steps with all other worker nodes
# BackUp
One option is to get all resources from all namespaces and move to YAML file:
```
kubectl get all --all-namespaces -o yaml > full_cluster.yaml
```
The other option is to backup etcd which stores all cluster state info.
Cluster data is in --data-dir in etcd.service or /etc/kubernetes/manifests/etcd.yaml (in case of etcd pod), typically /var/lib/etcd.
Here's how to upgrade when etcd is running as systemd service.
Check etcd major version first:
```
etcd version
```
Set ETCDCTL_API variable to major version and use snapshot save command:
```
ETCDCTL_API=3 etcdctl snapshot save /tmp/snapshot.db --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/etcd-server.crt --key=/etc/kubernetes/pki/etcd/etcd-server.key
```
Check etcd backup status:
```
ETCDCTL_API=3 etcdctl snapshot status /tmp/snapshot.db
```
To restore backup, stop kube-apiserver (as it depends on etcd):
```
service kube-apiserver stop
```
Then execute snapshot restore command:
```
ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --name=master \
     --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key \
     --data-dir /var/lib/etcd-from-backup \
     --initial-cluster=master=https://127.0.0.1:2380 \
     --initial-cluster-token=etcd-cluster-1 \
     --initial-advertise-peer-urls=https://127.0.0.1:2380 \
     snapshot restore /tmp/snapshot-pre-boot.db
```
Set --initial-cluster-token to token above and point --data-dir to new directory above in etcd.service.
Lastly, reload service daemon, restart etcd and start kube-apiserver again:
```
systemctl daemon-reload
service etcd restart
service kube-apiserver start
```
# How to create kubectl (admin) certificate to access kubernetes cluster
First, create private key as you normally would using openssl:
```
openssl genrsa -out rodrigo.key 2048
```
Then CSR request:
```
openssl req -new -key admin.key  -subj "/CN=rodrigo" -out rodrigo.csr
```
Copy contents of rodrigo.csr and encoded into base64 format (without spaces):
```
cat ~/.ssh/rodrigo.csr | base64 | tr -d '\n'
```
And paste it to YAML file:
```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: rodrigo
spec:
  groups:
  - system:authenticated
  usages:
  - digital signature
  - key encipherment
  - server auth
  request: <paste contents of "rodrigo.csr | base64" here
```
Reference: https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/#create-a-certificate-signing-request-object-to-send-to-the-kubernetes-api
Run YAML file:
```
kubectl create -f rodrigo.yaml
```
Check if request came in and is "pending":
```
kubectl get csr
```
Approve request:
```
kubectl certificate approve rodrigo
```
Kubernetes signs and generates a certificate using /etc/kubernetes/pki/ca.key and ca.crt or equivalent location from kube-controller systemd (or pod) --cluster-signing-cert-file and --cluster-signing-key-file.
Now copy certificate portion of 'kubectl get csr -o yaml', decode it and paste it to rodrigo.crt file:
```
kubectl get csr rodrigo -o jsonpath='{.status.certificate}' | base64 --decode > rodrigo.crt
```
