# Probe 検証ガイド

このフォルダでは Kubernetes の 3 種類の Probe（liveness / readiness / startup）と、失敗シナリオ再現用 Deployment 群を体験できます。順番に適用し、挙動・イベント・再起動回数・Endpoints 変化を観察してください。

対象ファイル:
- `livenessprobe.yaml`
- `readinessprobe.yaml`
- `startupprobe.yaml`
- `failure-scenarios.yaml`

今回は、別々のファイルに分けていますが、実際の運用では 1 つの Deployment 内に複数の Probe を組み合わせて使うことが多いです。

## 0. 前提コマンド例
```bash
# それぞれ適用 (個別検証時は必要なものだけでも可)
kubectl apply -f 06_Probe/livenessprobe.yaml
kubectl apply -f 06_Probe/readinessprobe.yaml
kubectl apply -f 06_Probe/startupprobe.yaml

kubectl get deploy -l app=probe-demo
kubectl get pods -l app=probe-demo -w
```

---
## 1. Liveness Probe 検証 (`livenessprobe.yaml`)
### 目的
コンテナが「生きているか（ハングしていないか）」を継続確認し、失敗し続けたら kubelet が **再起動** することを確認します。

### 期待される動作
- 正常時: Pod の `RESTARTS` は増えない。
- 異常時: liveness 失敗イベント (`Liveness probe failed`) が連続し、`RESTARTS` がインクリメント。再起動後は正常復帰。

### 観察コマンド
```bash
# Pod / イベント
kubectl get pod -l probe=liveness
kubectl describe pod <liveness Pod 名> | grep -i liveness -A2
```

### 失敗を人工的に起こす例
(単純化のため busybox でなく nginx のプロセスを一時停止)
```bash
# プロセスを STOP -> 応答停止扱い (一部環境で再現しにくい場合あり)　※ここでログに liveness probe の失敗が記録される
kubectl exec  <liveness Pod 名> -- sh -c 'pkill -STOP nginx'
# イベント観察
kubectl describe pod <liveness Pod 名> | grep -i liveness -A2
# 再開
kubectl exec  <liveness Pod 名> -- sh -c 'pkill -CONT nginx'
```

---
## 2. Readiness Probe 検証 (`readinessprobe.yaml`)
### 目的
**複数のPod環境**でReadiness Probeの真価を確認します。アプリケーションが一時的にトラフィックを処理できない状態になった時、Service のロードバランサーから**自動的に除外**され、他の健全なPodが処理を継続することを体験します。

### 期待される動作
- **複数Pod**: 通常時は全てのPodがEndpointsに登録され、負荷が分散される
- **1つのPodがNotReady**: 該当Podのみサービスから除外、他のPodは正常継続
- **Pod自体**: NotReadyでも `STATUS=Running` で再起動しない
- **自動復旧**: 問題解決後、自動的にロードバランサーに復帰

### 事前確認
```bash
# 複数Pod（3つ）が起動していることを確認
kubectl get pods -l probe=readiness
kubectl get endpoints readiness-sample-svc
# Endpoints に複数のPodのIPが表示されることを確認
```

### 観察コマンド
```bash
# 複数Pod環境での監視セットアップ
kubectl get pods -l probe=readiness -w &    # Pod状態監視
kubectl get endpoints readiness-sample-svc -w &    # ロードバランサー監視
```

### ロードバランシング除外テスト
特定のPodを意図的にNotReady状態にして、ロードバランサーからの除外を確認します。

```bash
# Step 1: 対象Podを選択（最初のPod）
TARGET_POD=$(kubectl get pod -l probe=readiness -o jsonpath='{.items[0].metadata.name}')
echo "対象Pod: $TARGET_POD"

# Step 2: 現在のEndpoints状況を確認
kubectl get endpoints readiness-sample-svc
kubectl get pods -l probe=readiness

# Step 3: 対象Podのサーバを停止（NotReady状態へ）
kubectl exec $TARGET_POD -- pkill nc || true

# Step 4: 変化を観察（10-20秒後に確認）
kubectl get pods -l probe=readiness
kubectl get endpoints readiness-sample-svc
```

### 💡 重要な観察ポイント
- **Endpoints の変化**: NotReady になったPodのIPが自動的に除外される
- **サービス継続**: 他の健全なPodがトラフィック処理を継続
- **無停止運用**: ユーザーからは問題のあるPodは見えない


---
## 3. Startup Probe 検証 (`startupprobe.yaml`)
### 目的
起動に時間がかかるコンテナで、起動完了前の一時的失敗で **早すぎる再起動が起きない** よう猶予 (grace period) を与える挙動を確認します。

### 期待される“良い”動作
- 起動中（startupProbe 成功前）は liveness / readiness の失敗が評価されない。
- startupProbe 成功後に liveness / readiness が通常稼働を開始。
- 許容時間 (failureThreshold * periodSeconds) を超えると `startupProbe failed` → 再起動。

### 観察
```bash
POD=$(kubectl get pod -l probe=startup -o jsonpath='{.items[0].metadata.name}')
# イベント
kubectl describe pod $POD | grep -i startup -A2
```

---
## 5. 期待される動作の判別基準まとめ
| Probe 種別 | 正常時 | 失敗時の挙動 | 再起動? | 主目的 | 複数Pod時の利点 |
|------------|--------|--------------|---------|--------|--------------| 
| liveness   | RESTARTS 増えない | イベント連続後 RESTARTS 増 | はい | ハング検知 & 自動復旧 | 異常Pod再起動中も他Podで継続サービス |
| readiness  | READY=1/1, Endpoints あり | READY=0/1, **Endpoints から除外** | いいえ | **ロードバランサーからの除外** | **無停止サービス継続** |
| startup    | 猶予内に成功し他 Probe 有効化 | 猶予超過で restart loop | はい (遅延中のみ) | 起動完了前の過剰再起動防止 | 段階的Pod起動で安定性向上 |

---
## 7. クリーンアップ
```bash
kubectl delete -f 06_Probe/failure-scenarios.yaml || true
kubectl delete -f 06_Probe/startupprobe.yaml || true
kubectl delete -f 06_Probe/readinessprobe.yaml || true
kubectl delete -f 06_Probe/livenessprobe.yaml || true
```