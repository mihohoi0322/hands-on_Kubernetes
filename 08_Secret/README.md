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
IDENTITY_NAME=wp-kv-mi
SA_NAMESPACE=default
SA_NAME=wordpress-app-sa
FC_NAME=wordpress-kv-fc
```

#### 2. AKS 機能有効化
```bash
az aks show -g $RG -n $AKS_NAME --query oidcIssuerProfile.issuerUrl -o tsv
az aks update -g $RG -n $AKS_NAME --enable-workload-identity          # 有効なら NOOP
az aks enable-addons -g $RG -n $AKS_NAME --addons azure-keyvault-secrets-provider
az aks get-credentials -g $RG -n $AKS_NAME
OIDC_ISSUER=$(az aks show -g $RG -n $AKS_NAME --query oidcIssuerProfile.issuerUrl -o tsv)
```

#### 3. ユーザー割当マネージド ID 作成 / 取得
```bash
az identity show -g $RG -n $IDENTITY_NAME --query '{name:name,clientId:clientId,principalId:principalId}' -o tsv
az identity create -g $RG -n $IDENTITY_NAME -l $LOC

IDENTITY_CLIENT_ID=$(az identity show -g $RG -n $IDENTITY_NAME --query clientId -o tsv)
IDENTITY_PRINCIPAL_ID=$(az identity show -g $RG -n $IDENTITY_NAME --query principalId -o tsv)
IDENTITY_RESOURCE_ID=$(az identity show -g $RG -n $IDENTITY_NAME --query id -o tsv)
echo "CLIENT_ID=$IDENTITY_CLIENT_ID"
```

#### 4. Key Vault 作成 / 取得
```bash
az keyvault show -n $KV_NAME -g $RG --query name -o tsv
az keyvault create -n $KV_NAME -g $RG -l $LOC --enable-rbac-authorization
KV_ID=$(az keyvault show -n $KV_NAME -g $RG --query id -o tsv)
```

#### 5. RBAC ロール割当
| ロール | 対象 | 用途 |
|--------|------|------|
| Key Vault Secrets User | UAMI | Pod からシークレット読み取り |
| Key Vault Secrets Officer | 操作者 (あなた) | シークレット登録・更新 |

```bash
CURRENT_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)
az role assignment create --assignee $IDENTITY_PRINCIPAL_ID --role "Key Vault Secrets User" --scope $KV_ID
az role assignment create --assignee $CURRENT_OBJECT_ID      --role "Key Vault Secrets Officer" --scope $KV_ID

az role assignment list --scope $KV_ID --assignee $CURRENT_OBJECT_ID --query '[].{principal:principalName,role:roleDefinitionName}' -o table
```

#### 6. フェデレーテッド資格情報 (Federated Credential) 作成
```bash
az identity federated-credential create -g $RG --identity-name $IDENTITY_NAME -n $FC_NAME --issuer $OIDC_ISSUER --subject system:serviceaccount:${SA_NAMESPACE}:${SA_NAME} --audience api://AzureADTokenExchange

az identity federated-credential list -g $RG --identity-name $IDENTITY_NAME -o table
```

#### 7. Key Vault シークレット投入
```bash
az keyvault secret set --vault-name $KV_NAME --name mysql-root-password --value 'rootpass123'
az keyvault secret set --vault-name $KV_NAME --name mysql-database      --value 'wordpress'
az keyvault secret set --vault-name $KV_NAME --name mysql-user          --value 'wpuser'
az keyvault secret set --vault-name $KV_NAME --name mysql-password      --value 'wppass123'
```

#### 8. マニフェスト準備
`keyvault-secretproviderclass.yaml` の ServiceAccount annotation を生成した `IDENTITY_CLIENT_ID` で置換:
```yaml
annotations:
  azure.workload.identity/client-id: <IDENTITY_CLIENT_ID>
```
SecretProviderClass 主要パラメータ (Workload Identity 時):
```yaml
parameters:
  usePodIdentity: "false"
  useVMManagedIdentity: "false"
  keyvaultName: "${KV_NAME}"  # 小文字
  tenantId: "$(az account show --query tenantId -o tsv)"
```

#### 9. デプロイ
```bash
kubectl apply -f 08_Secret/keyvault-secretproviderclass.yaml
kubectl apply -f 08_Secret/deployment-keyvault.yaml
```

#### 10. 動作検証
```bash
# Secret 同期確認
kubectl get secret wordpress-db-secret-kv -o yaml | head -n 20

# Frontend Pod 内ファイル
WP_POD=$(kubectl get pod -l app=wordpress,tier=frontend,source=kv -o jsonpath='{.items[0].metadata.name}')
kubectl exec $WP_POD -- ls -1 /mnt/kv-secrets

# 環境変数
kubectl exec $WP_POD -- env | egrep 'WORDPRESS_DB_|MYSQL_'
```

#### 11. トラブルシュート
| 症状 | 確認ポイント | 対処 |
|------|--------------|------|
| `Identity not found` | Federated Credential / annotation / tenantId | FC 作成/subject 再確認 & SA annotation client-id 一致 |
| `forbidden` | RBAC ロール不足 | Secrets User / Officer 付与状態再確認 (伝播待ち最大数分) |
| Secret 空 | Key Vault 名前/シークレット名 typo | `az keyvault secret show` で存在確認 |
| ローテ更新未反映 | Pod 再起動未実施 | `kubectl rollout restart` |
| 旧方式パラメータ混在 | useVMManagedIdentity=true など | 両方 false に統一 |



<!-- 上記で既に SecretProviderClass プレースホルダ / annotation / 適用方法を集約したため旧手順を整理 -->

### 差分ポイント（従来 Secret vs Key Vault）
| 項目 | 従来 Secret | Key Vault CSI |
|------|-------------|---------------|
| 保管場所 | etcd (K8s Secret) | Key Vault (同期 Secret も生成可) |
| ローテーション | 手動再適用 | KV 更新 + Pod 再起動 / 再マウント |
| 監査 | K8s 監査ログ | Key Vault 監査 / Azure Monitor |
| 暗号化キー管理 | etcd + KMS | Key Vault (HSM / 管理キー) |
| 利用方法 | envFrom | envFrom + ファイルマウント + 同期 Secret |

### シークレットローテーション (Key Vault)
既に前段の手順で AKS / Key Vault / Workload Identity / CSI Driver が構成済みである前提。パターン区分 (A/B) は排除し、最小手順のみを記載します。

#### ローテーション手順概要
1. Key Vault の対象シークレット値を更新 (新しいバージョンが作成される)
2. （任意）新バージョンが有効化されたことを確認
3. 対象 Deployment をローリング再起動（再マウント / Secret 同期）
4. Pod 内に反映されたことを環境変数 or マウントファイルで確認

#### 1. シークレット値更新 (例: MySQL パスワード)
```bash
az keyvault secret set --vault-name $KV_NAME --name mysql-password --value 'NEWpassw0rd!'
```

#### 2. バージョン確認 (任意)
```bash
az keyvault secret show --vault-name $KV_NAME --name mysql-password --query '{name:name,version:properties.version,updated:properties.updatedOn}' -o tsv
```

#### 3. ローリング再起動
```bash
kubectl rollout restart deploy/wordpress-mysql-kv
kubectl rollout restart deploy/wordpress-web-kv
kubectl rollout status deploy/wordpress-mysql-kv
kubectl rollout status deploy/wordpress-web-kv
```

#### 4. 反映確認
環境変数経由（例）:
```bash
kubectl exec $(kubectl get pod -l app=wordpress,tier=frontend,source=kv -o jsonpath='{.items[0].metadata.name}') -- env | grep MYSQL_PASSWORD
```
マウントファイル経由（必要なら）:
```bash
POD=$(kubectl get pod -l app=wordpress,tier=frontend,source=kv -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- cat /mnt/kv-secrets/mysql-password
```
> 補足: ローテーションを自動化する場合は
> - Azure Automation / Logic Apps で定期更新 + `kubectl rollout restart`
> - または Secrets Store CSI Driver Sync Secret + External Secrets Operator 組合せ
> を検討してください。

---
## 3. セキュリティ / ベストプラクティス
- 権限は最小限 (Key Vault の access policy / RBAC でシークレット取得のみ)
- 監査: Key Vault Diagnostic Setting → Log Analytics 送信
- Pod 内で不要ならファイルマウントを削り同期 Secret のみに一本化
- さらなる強化: Azure Policy で "Secret に平文パスワード禁止" のガバナンス

## 4. クリーンアップ
```bash
# シンプル版
kubectl delete -f 08_Secret/deployment.yaml || true
kubectl delete -f 08_Secret/secret.yaml || true

# Key Vault 版
kubectl delete -f 08_Secret/deployment-keyvault.yaml || true
kubectl delete -f 08_Secret/keyvault-secretproviderclass.yaml || true
```
（Key Vault 内シークレットは保持）

## 5. 次の発展例

- Azure Database for MySQL Flexible Server へ移行 (Managed Identity + Private Endpoint)
- Key Vault + Rotation ポリシーで自動パスワードローテーション
- External Secrets Operator との比較デモ
- HPA / VPA 追加でリソース自動調整

---
学習目的で平文 Secret と Key Vault 統合を並列配置しています。実運用では Key Vault / Workload Identity アプローチを標準化することを推奨します。
