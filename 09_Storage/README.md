# WordPress + Azure Files CSI (NFS) サンプル

このフォルダは WordPress の `wp-content` を Azurefiles で利用するサンプルです。

## 事前条件
0. 変数設定
```bash
RG=<RESOURCE_GROUP>
AKS_NAME=<AKS_CLUSTER>
STORAGE_ACCOUNT=<STORAGE_ACCOUNT_NAME>
```
1. AKS クラスター (Kubernetes バージョンがサポート範囲)
2. Azure Files CSI Driver を有効化する
     az aks update -g $RG -n $AKS_NAME --enable-file-driver
     kubectl get storageclass


## 適用手順 (動的プロビジョニング版)
```bash
kubectl apply -f 09_Storage/storageclass.yaml

# PVC (PV は StorageClass により自動作成)
kubectl apply -f 09_Storage/pvc.yaml
kubectl get pv,pvc

# Deployments + Services
kubectl apply -f 09_Storage/deployment.yaml
kubectl get pods -l app=wordpress

# 5. PVC マウント確認
POD=$(kubectl get pod -l app=wordpress,tier=frontend -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- mount | grep wp-content || true
kubectl exec $POD -- sh -c 'ls -al /var/www/html/wp-content'
```
