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

### 動作確認
```bash
# WordPress Pod の環境変数抜粋確認
POD=$(kubectl get pod -l app=wordpress,tier=frontend -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- env | grep WORDPRESS_DB_
```

---
## 2. Azure Key Vault CSI Driver 方式
Key Vault でシークレットを集中管理し、Pod 起動時に CSI ドライバでマウント + K8s Secret 同期。

### 前提
- AKS に Secrets Store CSI Driver + Azure Key Vault Provider 導入済み
- Workload Identity（推奨）または Managed Identity を設定済
- Key Vault に以下シークレット作成（例名）:
  - `mysql-root-password`
  - `mysql-database`
  - `mysql-user`
  - `mysql-password`

### `keyvault-secretproviderclass.yaml` 修正ポイント
| プレースホルダ | 内容 |
|----------------|------|
| `<YOUR_KEYVAULT_NAME>` | 対象 Key Vault 名 |
| `<YOUR_TENANT_ID>` | テナント ID |
| `<OPTIONAL-USER-ASSIGNED-IDENTITY-CLIENT-ID>` | ユーザー割当 MI 利用時 Client ID（不要なら空のまま可） |

`useVMManagedIdentity` を true にした場合: Node の (System または User Assigned) Managed Identity に Key Vault へのアクセス権 (RBAC か Access Policy) を付与。
Workload Identity を使う場合は `ServiceAccount` に以下例の annotation を追加:
```yaml
annotations:
  azure.workload.identity/client-id: <USER_ASSIGNED_MI_CLIENT_ID>
```

### 適用手順 (KV 版)
```bash
# SecretProviderClass + SA
kubectl apply -f 08_Secret/keyvault-secretproviderclass.yaml

# KV 版 Deployments + Services
kubectl apply -f 08_Secret/deployment-keyvault.yaml

# 同期された K8s Secret 確認
kubectl get secret wordpress-db-secret-kv -o yaml

# KV ファイルマウント確認
POD=$(kubectl get pod -l app=wordpress,tier=frontend,source=kv -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- ls /mnt/kv-secrets
```

### 差分ポイント
| 項目 | 従来 Secret | Key Vault CSI |
|------|-------------|---------------|
| 保管場所 | etcd (K8s Secret) | Azure Key Vault + 同期 Secret |
| ローテーション | 手動更新 & 再適用 | Key Vault 更新 → Pod 再起動 or 再マウントで反映 |
| 監査 | Kubernetes 監査ログ | Key Vault 監査 + Azure Monitor |
| 暗号化キー管理 | KMS(etcd) 依存 | Key Vault (HSM オプション可) |
| 利用方法 | envFrom | envFrom + ファイルマウント両方可 |

### ローテーション例
1. Key Vault で `mysql-password` を新値に更新
2. 新値バージョンをアクティブ化 (バージョン指定省略なら最新を取得)
3. Pod 再起動（RollingUpdate）
   ```bash
   kubectl rollout restart deploy/wordpress-mysql-kv
   kubectl rollout restart deploy/wordpress-web-kv
   ```
4. 新シークレット反映確認:
   ```bash
   kubectl exec $(kubectl get pod -l app=wordpress,tier=frontend,source=kv -o jsonpath='{.items[0].metadata.name}') -- env | grep MYSQL_PASSWORD
   ```

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
