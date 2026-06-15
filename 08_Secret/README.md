# WordPress + MySQL シークレット管理ガイド

このディレクトリでは、次の 2 パターンを扱います。

1. Kubernetes Secret（学習用途）
2. Azure Key Vault Provider for Secrets Store CSI Driver + Workload Identity

## ファイル一覧
| ファイル | 役割 |
|----------|------|
| `secret.yaml` | 平文(stringData) で DB/WordPress 用の Secret を定義（学習用途） |
| `deployment.yaml` | Secret を参照する MySQL / WordPress 構成 |
| `keyvault-secretproviderclass.yaml` | Workload Identity 用 `SecretProviderClass` |
| `deployment-keyvault.yaml` | `ServiceAccount` + Key Vault CSI 検証用 Pod |

---
## 1. Kubernetes Secret 方式

```bash
kubectl apply -f 08_Secret/secret.yaml
kubectl apply -f 08_Secret/deployment.yaml
kubectl get pods -l app=wordpress -w
kubectl get svc wordpress-web
```

動作確認:
```bash
POD=$(kubectl get pod -l app=wordpress,tier=frontend -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- env | grep WORDPRESS_DB_
kubectl get secret wordpress-db-secret
```

---
## 2. Key Vault CSI + Workload Identity 方式（Microsoft Learn 準拠）

公式:
- https://learn.microsoft.com/azure/aks/csi-secrets-store-driver
- https://learn.microsoft.com/azure/aks/csi-secrets-store-identity-access

### 前提
- AKS で OIDC Issuer / Workload Identity / Key Vault CSI アドオンが有効
- Key Vault に参照対象シークレットが作成済み

```bash
RG=<your-resource-group>
AKS_NAME=<your-aks-name>
UAMI=<your-user-assigned-managed-identity-name>
KEYVAULT_NAME=<your-keyvault-name>

az aks update -g $RG -n $AKS_NAME --enable-oidc-issuer --enable-workload-identity
az aks enable-addons -g $RG -n $AKS_NAME --addons azure-keyvault-secrets-provider

USER_ASSIGNED_CLIENT_ID=$(az identity show -g $RG -n $UAMI --query clientId -o tsv)
IDENTITY_TENANT=$(az aks show -g $RG -n $AKS_NAME --query identity.tenantId -o tsv)
KEYVAULT_SCOPE=$(az keyvault show -n $KEYVAULT_NAME --query id -o tsv)

# secret 参照なら Key Vault Secrets User を付与
az role assignment create --role "Key Vault Secrets User" --assignee $USER_ASSIGNED_CLIENT_ID --scope $KEYVAULT_SCOPE
```

### マニフェスト編集
`08_Secret/keyvault-secretproviderclass.yaml` と `08_Secret/deployment-keyvault.yaml` の以下を置換:
- `<USER_ASSIGNED_CLIENT_ID>`
- `<YOUR_KEYVAULT_NAME>`
- `<YOUR_TENANT_ID>`
- `<YOUR_SECRET_NAME>`

### デプロイ
```bash
kubectl apply -f 08_Secret/keyvault-secretproviderclass.yaml
kubectl apply -f 08_Secret/deployment-keyvault.yaml
```

### 動作確認
```bash
kubectl exec sc-demo-keyvault-csi -- ls -la /mnt/secrets-store
kubectl exec sc-demo-keyvault-csi -- cat "/mnt/secrets-store/<YOUR_SECRET_NAME>"
kubectl get secret keyvault-synced-secret -o yaml
```
