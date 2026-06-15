# AKS Hands-on Kubernetes ラボ集

このリポジトリは **Azure Kubernetes Service (AKS)** 上で基礎から拡張トピック (ストレージ / シークレット / Key Vault / Probe / スケーリング) まで一通り体験できるよう段階的フォルダ構成になっています。

> 学習用途のため「平文 Secret」「emptyDir の DB」など本番非推奨構成を含みます。実運用向け強化ポイントは各節末および本 README 末尾の "発展 / 改善提案" を参照してください。

---
## 目次
| ステップ | ディレクトリ | 内容 | 主目的 |
|----------|--------------|------|--------|
| 01 | `01_pod/` | 単一 Pod | 最小リソース適用フロー |
| 02 | `02_ReplicaSet/` | ReplicaSet | 自動再作成と selector/labels |
| 03 | `03_Deployment/` | Deployment | ローリング更新の基本 |
| 04 | `04_DeamonSet/` | DaemonSet (Fluentd) | 各 Node 常駐 / ログ収集概念 |
| 05a | `05_Job/` | Job / CronJob | バッチ / スケジュール実行 |
| 05b | `05_Service/` | Service (LoadBalancer) | 外部公開 (AKS LB) |
| 06 | `06_Probe/` | liveness / readiness / startup + 失敗シナリオ | ヘルスチェック設計 |
| 07 | `07_RequestLimits/` | Requests / Limits Pod | リソース管理と HPA 前提 |
| 08 | `08_Secret/` | Secret / Key Vault CSI | 機密情報の 2 手法比較 |
| 09 | `09_Storage/` | Azure Files CSI NFS (動的 PVC) + WordPress | ストレージ / 永続化の基礎 |
| 10 | `10_Scale/` | HPA (autoscaling/v2) | 水平方向オートスケール |

---
## 1. 前提環境 (ローカル)
- macOS / Linux シェル (zsh 等)
- `az` CLI 最新版
- `kubectl` / `kubelogin` / `helm` (任意)
- Azure サブスクリプション (Owner または必要 RBAC)

### インストール例 (参考)
```bash
brew update
brew install azure-cli kubectl helm
```

---
## 2. AKS クラスター準備
```bash
# 変数
RG=rg-aks-hol
LOC=japaneast
AKS=aks-hol

az group create -n $RG -l $LOC
# Workload Identity / OIDC / Key Vault Provider / Azure Files CSI を最初から有効化 (再作成コスト削減)
az aks create -g $RG -n $AKS \
  --node-count 2 --node-vm-size Standard_B4ms \
  --enable-oidc-issuer --enable-workload-identity \
  --enable-addons azure-keyvault-secrets-provider \
  --enable-file-driver \
  --generate-ssh-keys

# 取得 & コンテキスト設定
az aks get-credentials -g $RG -n $AKS --overwrite-existing

# 動作確認
kubectl get nodes
```
> 注: 既存 AKS に追加する場合は `az aks update --enable-workload-identity --enable-oidc-issuer` / `--enable-file-driver` / `--enable-addons azure-keyvault-secrets-provider` を個別実行。

### メトリクス (HPA 用)
AKS は既定で metrics-server がデプロイされています。`kubectl top nodes` で確認できない場合は数分待機。

---
## 3. 学習用 Namespace (推奨)
```bash
kubectl create namespace hol || true
# 以降: -n hol を付けるか、KUBECONFIG context を namespace=hol に切替
kubectl config set-context --current --namespace=hol
```
> 現在のマニフェストは `metadata.namespace` を指定していないため default namespace に作成されます。ハンズオンの切り離し・クリーンアップを容易にするため namespace 化を推奨。

---
## 4. 進め方ガイド (ハイレベル手順)
1. 01〜03: 基本オブジェクト (Pod→ReplicaSet→Deployment)
2. 05_Service の Service を 03 の Deployment と組み合わせ外部公開
3. 06_Probe: 各 Probe と失敗シナリオ観察
4. 07_RequestLimits: Requests/Limits の出力と HPA 前提理解
5. 08_Secret: 平文 Secret → Key Vault CSI へ進化 (Workload Identity)
6. 09_Storage: Azure Files CSI (NFS) 動的 PVC (WordPress `/var/www/html`)
7. 10_Scale: HPA によるスケール挙動
8. Optional: 改善課題 (下記 "発展 / 改善提案") を自分で拡張

---
## 5. 主要トピック別メモ
### シークレット管理 (08_Secret)
| 手法 | メリット | 課題 | 本番推奨度 |
|------|----------|------|-------------|
| 平文 Kubernetes Secret | 手軽 / すぐ試せる | Git 流出リスク / 手動ローテーション | 低 |
| Key Vault CSI + Workload Identity | 中央管理 / ローテーション容易 / RBAC | 初期構築少し複雑 | 高 |

### ストレージ (09_Storage)
| 種別 | 用途 | 特徴 | 注意 |
|------|------|------|------|
| Azure Files CSI NFS (本サンプル) | 読み書き共有 / WordPress 共有コンテンツ | RWX / 動的 PVC 対応 | NFS 制約・レイテンシに注意 |
| Azure Files | 読み書き共有 / POSIX 互換 | 標準 RWX | 性能要件に応じ SKU 選択 |
| Azure Disk | 単一 Pod 高性能 RW | 高 IOPS / 低レイテンシ | RWX 不可 (マルチアタッチは制限) |

### Probe 設計 (06_Probe)
- readiness: トラフィック振り分け制御
- liveness: 自己修復
- startup: 起動猶予 (遅延初期化)
- 失敗シナリオで観察: `kubectl describe pod` / Endpoints 変化 / RESTARTS 変化

### HPA (10_Scale)
- autoscaling/v2 で CPU + メモリ複合ターゲット
- `requests` が基準 (Utilization%)
- `behavior` で scaleUp/Down ポリシー制御

---
## 6. 代表的な動作確認コマンド
```bash
# ラベル指定で監視
a=app=probe-demo; kubectl get pods -l $a -w &
# Pod イベント
target=$(kubectl get pod -l probe=liveness -o jsonpath='{.items[0].metadata.name}'); kubectl describe pod $target | tail -n 30
# HPA 監視
kubectl get hpa -w
# トップ
kubectl top pods -l app=hpa-demo
# WordPress 外部 IP
kubectl get svc -l app=wordpress -w
```

---
## 7. クリーンアップ (一括例)
```bash
# 注意: default / hol 両方に作成している場合は namespace を付けて再実行
for d in 10_Scale 09_Storage 08_Secret 07_RequestLimits 06_Probe 05_Service 05_Job 04_DeamonSet 03_Deployment 02_ReplicaSet 01_pod; do 
  kubectl delete -f $d --ignore-not-found --recursive || true
  # ディレクトリ直下のみ multi-doc でない場合 `-f $d/*.yaml` の方が明示的
done
# Azure Files 用 PVC/SC (順序注意)
kubectl delete -f 09_Storage/pvc.yaml --ignore-not-found || true
kubectl delete -f 09_Storage/storageclass.yaml --ignore-not-found || true
# Key Vault CSI 追加リソース
kubectl delete -f 08_Secret/deployment-keyvault.yaml --ignore-not-found || true
kubectl delete -f 08_Secret/keyvault-secretproviderclass.yaml --ignore-not-found || true
```
(リソースグループごと削除する場合: `az group delete -n $RG -y --no-wait`)

---
## 8. セキュリティ / ベストプラクティス抜粋
| 項目 | 改善ポイント |
|------|--------------|
| Secret 平文 | GitHub への push 前に SOPS / Key Vault 化 or 削除 |
| MySQL 永続化 | `emptyDir` → Azure Disk PVC / Azure Database for MySQL (PaaS) |
| Workload Identity | `wordpress-app-sa` に適切な annotation + Federated Credential 設定 |
| ストレージキー | Key Vault / Storage SAS / Managed Identity へ移行 |
| Namespace 分離 | `hol` や `teamA` など名前空間化 + RBAC サンプル追加 |
| Network | Private Cluster / NSG / Azure Firewall / WAF (Ingress Controller + AGIC) |
| 監視 | Container Insights / Azure Monitor Managed Prometheus + Alert 追加 |

---
## 9. 発展 / 改善提案 (次の練習)
1. Ingress Controller (NGINX or AGIC) + HTTPS (Let’s Encrypt / Azure Key Vault 証明書)
2. Azure Files Premium による WordPress コンテンツ永続化比較
3. MySQL → Azure Database for MySQL (Flexible Server) + Private Endpoint
4. External Secrets Operator 版 (Key Vault → Secret 同期差分比較)
5. PodDisruptionBudget / Topology Spread Constraints / Affinity 設定
6. VPA (Vertical Pod Autoscaler) or KEDA (イベント駆動スケール) 追加
7. RBAC: 読み取り専用 Role + 開発者向け namespace 制限サンプル
8. GitOps (Flux / Argo CD) でこのリポジトリを継続適用
9. Azure Policy による強制ガバナンス (例: 要求: livenessProbe 必須 / no privileged)
10. Chaos Engineering (Azure Chaos Studio) で Probe / HPA の耐性検証

---
## 10. ラボ品質チェック (現状まとめ)
| 項目 | 状態 |
|------|------|
| 基本 Pod/ReplicaSet/Deployment | OK |
| Service (LB) | OK (annotations 追加余地) |
| Job/CronJob | OK |
| Probes + Failure Scenarios | OK (詳細 README) |
| Requests/Limits | OK (07 + 各 Deployment) |
| Secret (平文) | OK (学習用途) / 本番は非推奨 |
| Key Vault CSI | OK (placeholder 要編集) |
| Azure Files CSI NFS 動的 PVC | OK (学習用 / RWX 注意) |
| HPA (CPU+Memory) | OK |
| Workload Identity | OK (Key Vault 検証用マニフェストで設定済み) |
| 永続 MySQL | 未実装 (emptyDir) |
| Namespace / RBAC | 未導入 (拡張余地) |
| 監視 / ログ統合 | 未記載 (Azure Monitor 手順追加余地) |

---
## 11. 既知のプレースホルダ / 編集必須箇所
| ファイル | 置き換える値 |
|---------|--------------|
| `08_Secret/deployment-keyvault.yaml` | `<USER_ASSIGNED_CLIENT_ID>` |
| `08_Secret/keyvault-secretproviderclass.yaml` | `<USER_ASSIGNED_CLIENT_ID>` `<YOUR_KEYVAULT_NAME>` `<YOUR_TENANT_ID>` `<YOUR_SECRET_NAME>` |

---
## 12. 貢献 / 変更方針
- 新しいサンプルは番号を増分 (`11_...`) で追加
- multi-doc YAML の場合は先頭に用途コメント
- 機密値はコミットしない (ダミー or placeholder) 

PR テンプレ (推奨):
```
### 目的

### 変更点
- [ ] 新規フォルダ `11_XXXX`
- [ ] 既存 README 更新

### 動作確認
(コマンド / スクリーンショット)
```

---
**Happy AKS Hacking!** さらに発展させたいトピックがあれば Issue / PR 歓迎です。
