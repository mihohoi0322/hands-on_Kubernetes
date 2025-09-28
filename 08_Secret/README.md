# WordPress + MySQL シークレット管理ガイド

ここでは 2 通りのシークレット管理を実行します。

1. 一般的な Secret
2. Azure Key Vault CSI Driver を用いた Key Vault 連携

## ファイル一覧
| ファイル | 役割 |
|----------|------|
| `secret.yaml` | 平文(stringData) で DB/WordPress 用環境変数を定義（学習用） |
| `deployment.yaml` | Secret を参照する MySQL / WordPress の基本構成 |
| `keyvault-secretproviderclass.yaml` | Key Vault CSI Driver の `SecretProviderClass` + `ServiceAccount` |
| `deployment-keyvault.yaml` | Key Vault から同期された Secret を利用する MySQL / WordPress 構成 |

---
## 1. シンプルな Kubernetes Secret 方式

### 適用手順
```bash
# Secret / Deployment / Service 適用
kubectl apply -f 08_Secret/secret.yaml
kubectl apply -f 08_Secret/deployment.yaml

# Pod 監視
kubectl get pods -l app=wordpress -w

# 外部公開 Service 確認
kubectl get svc wordpress-web
```

### 動作確認 & Secret 正常性検証

まず WordPress Pod に環境変数が渡っていることを確認します。
```bash
# WordPress Pod 名取得 (フロントエンド)
POD=$(kubectl get pod -l app=wordpress,tier=frontend -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- env | grep WORDPRESS_DB_
```

#### 1. Secret の存在とキーフォーマット確認
```bash
# Secret 一覧に存在するかを確認
kubectl get secret wordpress-db-secret

# 中身のキー一覧 (値は base64 エンコード済み)
kubectl describe secret wordpress-db-secret | grep -E 'MYSQL_|WORDPRESS_'
```

#### 2. 個別キーのデコード検証 (値を画面に出して良い学習環境でのみ)
```bash
# 代表的なキーをデコード
kubectl get secret wordpress-db-secret -o jsonpath='{.data.MYSQL_DATABASE}' | base64 -d; echo
kubectl get secret wordpress-db-secret -o jsonpath='{.data.MYSQL_USER}' | base64 -d; echo
```

#### 3. ブラウザ確認
以下にアクセスし WordPress インストール画面 (言語選択 / セットアップ) が表示されれば成功。
```
http://<EXTERNAL-IP>/wp-admin/install.php
```
`<EXTERNAL-IP>` は `kubectl get svc wordpress-web -o jsonpath='{.status.loadBalancer.ingress[0].ip}'` 等で取得。

---
## 2. Azure Key Vault CSI Driver 方式（Workload Identity 準拠版）
公式手順: https://learn.microsoft.com/azure/aks/csi-secrets-store-identity-access (Microsoft Entra Workload Identity pivot)

Key Vault のシークレットを Pod 起動時に CSI Driver でファイル & Kubernetes Secret に同期し、環境変数 (envFrom) でも利用します

### 手順概要
1. 変数定義
2. AKS の OIDC Issuer & Workload Identity & Key Vault CSI アドオン有効化
3. ユーザー割当マネージド ID (UAMI) 作成 / 取得
4. Key Vault 作成 / 取得
5. RBAC ロール割当 (UAMI=Secrets User / 操作者=Secrets Officer)
6. フェデレーテッド資格情報 (Federated Credential) 作成
7. Key Vault にシークレット投入
8. マニフェスト準備 (ServiceAccount annotation / SecretProviderClass)
9. デプロイ (SecretProviderClass → Deployment)
10. 動作検証 (Secret/ファイル/環境変数)
11. トラブルシュート

---
#### 1. 変数定義
```bash
RG=<your-resource-group>
AKS_NAME=<your-aks-name>
LOC=$(az aks show -g $RG -n $AKS_NAME --query location -o tsv)
KV_NAME=<your-keyvault-name>
```

#### 2. AKS 機能有効化
```bash
az aks show -g $RG -n $AKS_NAME --query oidcIssuerProfile.issuerUrl -o tsv
az aks update -g $RG -n $AKS_NAME --enable-workload-identity
az aks enable-addons -g $RG -n $AKS_NAME --addons azure-keyvault-secrets-provider
```

#### セットアップ
```bash
az provider register -n Microsoft.ServiceLinker
az provider register -n Microsoft.KubernetesConfiguration
```

#### 4. Key Vault 作成 / 取得
```bash
az keyvault show -n $KV_NAME -g $RG --query name -o tsv
az keyvault create -n $KV_NAME -g $RG -l $LOC --enable-rbac-authorization
KV_ID=$(az keyvault show -n $KV_NAME -g $RG --query id -o tsv)

# Secret の権限を付与する
MY_OBJECT_ID=$(az ad signed-in-user show --query "id" --output tsv)
echo "My Object ID: $MY_OBJECT_ID"
az role assignment create --role "Key Vault Secrets Officer" --assignee $MY_OBJECT_ID --scope /subscriptions/$(az account show --query id --output tsv)/resourcegroups/${RG}/providers/Microsoft.KeyVault/vaults/${KV_NAME}
```

#### 5. RBAC ロール割当
| ロール | 対象 | 用途 |
|--------|------|------|
| Key Vault Secrets User | UAMI | Pod からシークレット読み取り |
| Key Vault Secrets Officer | 操作者 (あなた) | シークレット登録・更新 |



#### 7. Key Vault シークレット投入
```bash
SECRET_NAME="krmt-test"
VALUE=""

az keyvault secret set --vault-name $KV_NAME --name $SECRET_NAME --value $VALUE
```

#### サービスコネクタを使って、AKS でサービス接続を作成する
```bash
CONN_NAME=""
az aks connection create keyvault --connection $CONN_NAME --resource-group $RG --name $AKS_NAME --target-resource-group $RG --vault $KV_NAME --enable-csi --client-type none
```

#### 変数準備
```bash
# Service Connectorで設定された値を確認
az aks connection list-configuration --resource-group $RG --name $AKS_NAME --connection "keyvault-connection"

# CSI DriverのマネージドアイデンティティのクライアントIDを取得
export AZURE_KEYVAULT_CLIENTID=$(az aks show --resource-group $RG --name $AKS_NAME --query 'addonProfiles.azureKeyvaultSecretsProvider.identity.clientId' --output tsv)

# テナントIDを取得
export AZURE_KEYVAULT_TENANTID=$(az account show --query tenantId --output tsv)

echo "Key Vault Name: $KV_NAME"
echo "Client ID: $AZURE_KEYVAULT_CLIENTID"
echo "Tenant ID: $AZURE_KEYVAULT_TENANTID"
echo "Secret Name: $SECRET_NAME"
```

#### 9. デプロイ
```bash
cat <<EOF > ./08_Secret/secretproviderclass.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: sc-demo-keyvault-csi
  namespace: default
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: "${AZURE_KEYVAULT_CLIENTID}"
    keyvaultName: "${KV_NAME}"
    tenantId: "${AZURE_KEYVAULT_TENANTID}"
    objects: |
      array:
        - |
          objectName: ${SECRET_NAME}
          objectType: secret
          objectVersion: ""
EOF

ファイルが作成されたのち、実行
```bash
kubectl apply -f 08_Secret/keyvault-secretproviderclass.yaml
kubectl apply -f 08_Secret/deployment-keyvault.yaml
```

#### 10. 動作検証
```bash
# Secret 同期確認
kubectl exec sc-demo-keyvault-csi -- ls -la /mnt/secrets-store

# 内容確認
kubectl exec sc-demo-keyvault-csi -- cat /mnt/secrets-store/krmt-test
```
