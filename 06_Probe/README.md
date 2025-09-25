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

### 期待される“良い”動作
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
コンテナが「トラフィックを受ける準備ができているか」を判定し、未準備なら Service の Endpoints から **外される**（再起動はしない）ことを確認します。

### 期待される“良い”動作
- Pod ステータス: READY が 0/1 の間でも `STATUS=Running` であり再起動しない。
- `kubectl get endpoints readiness-sample-svc` で Endpoints が空になる (NotReady 時)。
- 準備完了後は READY 1/1 となり Endpoints に再登録。

### 観察コマンド
```bash
POD=$(kubectl get pod -l probe=readiness -o jsonpath='{.items[0].metadata.name}')
# Pod 状態ウォッチ
kubectl get pod $POD -w &
# Endpoints 変化ウォッチ
kubectl get endpoints readiness-sample-svc -w &
```

### 一時的に NotReady にする
Deployment 内コンテナは `readinessProbe (tcpSocket:8080)` を持ち、サーバ停止で失敗します。
```bash
# サーバ停止 (nc の簡易サーバを終了)
kubectl exec $POD -- pkill nc || true
# READY 0/1 になるはず
# 復帰 (簡易 HTTP 応答ループ再起動)
kubectl exec $POD -- sh -c 'while true; do printf "HTTP/1.1 200 OK\r\nContent-Length:2\r\n\r\nok" | nc -l -p 8080; done &' 
```

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

### 失敗バージョンを試したい場合
`startupprobe.yaml` 内 `sleep 40` を `sleep 60` に変更し再適用 (許容 50 秒設定)。
```bash
# 編集後
kubectl apply -f 06_Probe/startupprobe.yaml
kubectl describe pod $POD | grep -i startup -A4
```

---
## 4. Failure シナリオ集 (`failure-scenarios.yaml`)
適用:
```bash
kubectl apply -f 06_Probe/failure-scenarios.yaml
```

### 4-1. readiness-toggle
- `readinessProbe` (exec) は `/tmp/force-not-ready` があると失敗。
- 期待: ファイル作成で READY 0/1 & Endpoints 0、削除で復帰。
```bash
RP=$(kubectl get pod -l scenario=readiness-toggle -o jsonpath='{.items[0].metadata.name}')
kubectl exec $RP -- sh -c 'touch /tmp/force-not-ready'
kubectl get pod $RP
kubectl exec $RP -- rm /tmp/force-not-ready
```

### 4-2. liveness-toggle
- heartbeat ファイルの更新停止 or `/tmp/force-dead` 作成で liveness 失敗 → 再起動。
```bash
LP=$(kubectl get pod -l scenario=liveness-toggle -o jsonpath='{.items[0].metadata.name}')
# 強制失敗
kubectl exec $LP -- sh -c 'touch /tmp/force-dead'
# イベント / 再起動確認
kubectl describe pod $LP | grep -i liveness -A2
```

### 4-3. startup-failure
- 起動 sleep が startupProbe 許容時間を超え、`startupProbe failed` が繰り返される。
```bash
SP=$(kubectl get pod -l scenario=startup-failure -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $SP | grep -i startup -A2
```

---
## 5. 期待される“良い”動作の判別基準まとめ
| Probe 種別 | 正常時 | 失敗時の挙動 | 再起動? | 主目的 |
|------------|--------|--------------|---------|--------|
| liveness   | RESTARTS 増えない | イベント連続後 RESTARTS 増 | はい | ハング検知 & 自動復旧 |
| readiness  | READY=1/1, Endpoints あり | READY=0/1, Endpoints 0 | いいえ | トラフィック遮断 (再起動は不要) |
| startup    | 猶予内に成功し他 Probe 有効化 | 猶予超過で restart loop | はい (遅延中のみ) | 起動完了前の過剰再起動防止 |

Failure シナリオは上記を意図的に再現し、しきい値設計を検証するための補助です。

---
## 6. よくある設計アンチパターン
| アンチパターン | 問題点 | 改善 |
|----------------|--------|------|
| readiness と liveness を同一スクリプトで流用 | 一時的遅延で再起動連発 (不必要) | readiness をより頻度高 / liveness は安定パス |
| 初期化が遅いのに startupProbe 未使用 | 不要な CrashLoop / cold start 延長 | startupProbe 追加で猶予 |
| timeoutSeconds が過度に短い | ネットワーク一時遅延で誤検知 | 平均 + 余裕 (p95 レイテンシ) で設定 |
| failureThreshold=1 (過敏) | 瞬間的スパイクで flapping | 2〜3 程度を基本線 |

---
## 7. クリーンアップ
```bash
kubectl delete -f 06_Probe/failure-scenarios.yaml || true
kubectl delete -f 06_Probe/startupprobe.yaml || true
kubectl delete -f 06_Probe/readinessprobe.yaml || true
kubectl delete -f 06_Probe/livenessprobe.yaml || true
```

## 8. 次の発展案
- HTTP / gRPC Health エンドポイント実装例追加
- canary デプロイと絡めた readiness 遅延テスト
- Chaos Engineering (pod-kill / network-delay) を加えた自動検証

以上で基本から失敗再現まで一通り学習できます。必要があれば追加シナリオをリクエストしてください。
