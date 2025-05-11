# Azure Kubernetes Service 上での n8n 運用ガイド

## 1. デプロイ概要
| リソース | 値 |
|---|---|
| Namespace | `n8n` |
| n8n イメージ | `n8nio/n8n:latest` *(manifest でタグ未指定のため最新安定版)* |
| Postgres イメージ | `postgres:11` |
| 外部公開方法 | Service type `LoadBalancer` |
| n8n URL (現状) | `http://74.176.128.250:5678/` |
| データベース | Postgres (DB 名: `n8n`) |
| PVC 容量 | Postgres: **300 GiB**, n8n ファイル: **2 GiB** |
| Pod リソース (n8n) | `requests.memory=250Mi`, `limits.memory=500Mi` |
| Pod リソース (Postgres) | `requests.cpu=1`, `requests.memory=2Gi`, `limits.cpu=4`, `limits.memory=4Gi` |

## 2. 構成ファイル一覧
| ファイル | 用途 |
|---|---|
| `namespace.yaml` | Namespace `n8n` の定義 |
| `postgres-secret.yaml` *(CLI で上書き生成)* | DB 認証情報 (ただし CLI で生成したため当ファイルは未使用) |
| `postgres-configmap.yaml` | 非 root ユーザ作成用 init スクリプト |
| `postgres-claim0-persistentvolumeclaim.yaml` | Postgres 用 PVC (300 GiB) |
| `n8n-claim0-persistentvolumeclaim.yaml` | n8n ファイル保存用 PVC (2 GiB) |
| `postgres-deployment.yaml` / `postgres-service.yaml` | Postgres 本体と ClusterIP Service |
| `n8n-deployment.yaml` / `n8n-service.yaml` | n8n 本体と LoadBalancer Service |

## 3. シークレット
```
Secret 名         : postgres-secret
キー               : POSTGRES_USER=postgres
                     POSTGRES_PASSWORD=<ランダム生成>
                     POSTGRES_NON_ROOT_USER=n8n
                     POSTGRES_NON_ROOT_PASSWORD=<ランダム生成>
```
ランダム生成したパスワードはローカル PowerShell 変数 `$POST_PASS` に保持しています。忘れた場合は以下で取得可能です。
```powershell
kubectl get secret postgres-secret -n n8n -o jsonpath="{.data.POSTGRES_PASSWORD}" | base64 -d
```

n8n 用の `n8n-secret` は現在 `dummy` プレースホルダーのみ。Basic 認証等を追加する場合は次のコマンド例で追記します。
```powershell
kubectl create secret generic n8n-secret \
  --from-literal=N8N_BASIC_AUTH_ACTIVE=true \
  --from-literal=N8N_BASIC_AUTH_USER=admin \
  --from-literal=N8N_BASIC_AUTH_PASSWORD=<password> \
  -n n8n --dry-run=client -o yaml | kubectl apply -f -
```

## 4. 運用タスク
### 4.1 スケーリング
* n8n の水平スケール
```powershell
kubectl scale deployment n8n -n n8n --replicas=<数>
```
* Postgres は Stateful なため、スケールアウト非推奨。必要に応じて vCPU / メモリを増やすか、マネージド DB へ移行検討。

### 4.2 アップグレード
1. n8n
```powershell
# 例: 1.93.0 へ更新
kubectl set image deployment/n8n n8n=n8nio/n8n:1.93.0 -n n8n
```
2. Postgres はメジャーバージョン互換に注意して `postgres-deployment.yaml` のイメージタグを編集 → `kubectl apply -f`。

### 4.3 バックアップ
* **Postgres**: Azure Disk Snapshot / `pg_dumpall` を cronjob 化。PVC 名: `postgresql-pv`。
* **n8n 設定ファイル (.n8n)**: `n8n-claim0` PVC を同様に Snapshot。

### 4.4 監視 / ログ
```powershell
# リアルタイムログ確認
kubectl logs -f deploy/n8n -n n8n
kubectl logs -f deploy/postgres -n n8n
```
* Azure Monitor Container Insights を有効化するとメトリクスとログをポータルで一元管理可能。

### 4.5 Secrets ローテーション
1. 新しいパスワードで `postgres-secret` を上書き再作成
2. `n8n` Deployment をローリング再起動
```powershell
kubectl rollout restart deployment n8n -n n8n
```

### 4.6 容量拡張
```powershell
# 例: Postgres PVC を 500Gi へ拡大
kubectl patch pvc postgresql-pv -n n8n -p '{"spec":{"resources":{"requests":{"storage":"500Gi"}}}}'
```
Azure Disk は自動リサイズ完了後に Pod を再起動すると新容量を利用可能。

## 5. セキュリティ / 公開設定
1. **TLS**: Ingress + cert-manager + Let's Encrypt を推奨。
2. **DNS**: サブドメイン `n8n.<yourdomain>` の A レコードを外部 IP `74.176.128.250` に向ける。
3. **IP 制限**: Azure Firewall / NSG で 5678/TCP へのアクセスを許可 IP のみに制限可能。

## 6. よくある操作コマンド集
```powershell
# Pod/Service 状況
kubectl get all -n n8n

# Deployment のロールバック
kubectl rollout history deployment/n8n -n n8n
kubectl rollout undo deployment/n8n -n n8n --to-revision=<rev>

# Postgres CLI に入る
kubectl exec -it deploy/postgres -n n8n -- psql -U n8n -d n8n
```

## 7. トラブルシューティング
| 症状 | チェックポイント |
|---|---|
| n8n 画面が 502/接続不可 | `kubectl get pods -n n8n`, Pod 再起動ループ有無を確認 |
| DB 接続エラー | `postgres-secret` の NON_ROOT_PASSWORD と n8n デプロイ env の整合性 |
| ディスク Full | `kubectl describe pvc` で使用量を確認し、容量拡張 |

---
以上を参考に日常運用・バージョンアップ・障害対応を行ってください。 