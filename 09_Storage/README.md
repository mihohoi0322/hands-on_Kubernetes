# WordPress + Azure Blob Storage (CSI) サンプル

このフォルダは WordPress の `wp-content` を Azure Blob Storage (blobfuse2) で利用するサンプルです。

## 構成ファイル
| ファイル | 役割 |
|----------|------|
| `secret.yaml` | DB 接続情報 + Blob 用 `azurestorageaccountname` / `azurestorageaccountkey` |
| `storageclass.yaml` | blob.csi.azure.com 用 StorageClass (container `wpcontent`) |
| `pv-pvc.yaml` | (動的) PersistentVolumeClaim のみ。StorageClass により PV 自動作成 |
| `deployment.yaml` | MySQL / WordPress Deployments & Services。WordPress が PVC を `/var/www/html/wp-content` にマウント |

## 事前条件
0. 変数設定
    ```bash
    RG=<RESOURCE_GROUP>
    CLUSTER=<AKS_CLUSTER>
    STORAGE_ACCOUNT=<STORAGE_ACCOUNT_NAME>
    ```
1. AKS クラスター (Kubernetes バージョンがサポート範囲)
2. Blob CSI Driver が有効化されていること
   - AKS アドオン:
     ```bash
     az aks update -g $RG -n $CLUSTER --enable-blob-driver
     ```
   - もしくは公式マニフェスト / Helm によるインストール。
3. Azure Storage アカウント + コンテナ `wpcontent` を作成:
   ```bash
   az storage account create -g $RG -n $STORAGE_ACCOUNT --sku Standard_LRS
   az storage container create --account-name $STORAGE_ACCOUNT -n wpcontent
   az storage account keys list -g $RG -n $STORAGE_ACCOUNT --query [0].value -o tsv
   ```
4. `secret.yaml` 内の `<YOUR_STORAGE_ACCOUNT_NAME>` / `<YOUR_STORAGE_ACCOUNT_KEY>` を実値に置換。

## 適用手順 (動的プロビジョニング版)
```bash
# 1. Secret (DB+Blob 認証情報)
kubectl apply -f 09_Storage/secret.yaml

# 2. StorageClass
yaml=09_Storage/storageclass.yaml; kubectl apply -f $yaml

# 3. PVC (PV は StorageClass により自動作成)
kubectl apply -f 09_Storage/pv-pvc.yaml
kubectl get pvc wp-blob-pvc -o wide
kubectl get pv | grep blob-fuse

# 4. Deployments + Services
kubectl apply -f 09_Storage/deployment.yaml
kubectl get pods -l app=wordpress -w

# 5. PVC マウント確認
POD=$(kubectl get pod -l app=wordpress,tier=frontend -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- mount | grep wp-content || true
kubectl exec $POD -- sh -c 'ls -al /var/www/html/wp-content'
```

## 動作ポイント
- 旧: 静的 PV + PVC → 新: PVC のみ (StorageClass `blob-fuse` が動的に PV を作成)
- StorageClass に `secretName` / `secretNamespace` を追加しストレージアカウントキーを参照
- ReclaimPolicy は学習用に `Delete`。データ保持したい場合は `Retain` に変更
- WordPress の `wp-content` は PVC (`wp-blob-pvc`) を介して Blobfuse2 マウント

## クリーニング (順序: アプリ → PVC → SC → Secret)
```bash
kubectl delete -f 09_Storage/deployment.yaml || true
kubectl delete -f 09_Storage/pv-pvc.yaml || true   # PVC 削除 (PV も Delete ポリシーで削除)
kubectl delete -f 09_Storage/storageclass.yaml || true
kubectl delete -f 09_Storage/secret.yaml || true
```
(ReclaimPolicy=Delete のため PV 削除でボリュームも解放。保持したい場合は Retain に変更)

## トラブルシュート
| 症状 | 確認 / 対処 |
|------|-------------|
| PVC が Pending | PV と accessModes / storageClassName / size が一致するか |
| Pod Mount エラー (Permission) | mountOptions の `-o allow_other`、secret キー名綴り確認 |
| パフォーマンス低下 | Blob は高 IOPS ワークロードに不向き。Azure Files Premium / Disk 検討 |
| 一部プラグイン書き込み失敗 | Blobfuse2 の POSIX 制限。Azure Files へ移行検討 |

## 発展 (本番向け)
- Azure Files (Premium) で RWX 高互換ストレージ
- Key Vault CSI Driver でストレージキー非開示化
- MySQL を Azure Database for MySQL Flexible Server へ分離 (Managed Identity + Private Link)
- CDN / Front Door で静的アセット配信

## 参考
- Azure AKS Storage Concepts: https://learn.microsoft.com/azure/aks/concepts-storage
- Azure Blob CSI Driver: https://learn.microsoft.com/azure/aks/azure-blob-csi

---
