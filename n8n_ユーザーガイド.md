# n8n ユーザーガイド (AKS 版)

> 本ドキュメントは Azure Kubernetes Service 上にデプロイされた n8n を **はじめて使う方向け** のクイックスタートです。画面遷移や用語は n8n v1.x 系を前提に記載しています。

---

## 1. アクセスと初期セットアップ
1. ブラウザで以下 URL を開く  
   `http://74.176.128.250:5678/`
2. 初回アクセス時は **Owner アカウント作成** 画面が表示されるので、メールアドレスとパスワードを入力して登録します。
3. 以降はそのメールアドレスでログインします。

> ⚠️ **重要:** 現在 `N8N_SECURE_COOKIE` が `true` （デフォルト）のため、HTTPS 接続でないとログインできません。一時的に HTTP で試す場合は、次の手順で `n8n-secret` に `N8N_SECURE_COOKIE=false` を追加してください。（非推奨）
> ```powershell
> kubectl patch secret n8n-secret -n n8n -p '{ "data": { "N8N_SECURE_COOKIE": "ZmFsc2U=" } }' # "false" の Base64 値
> kubectl rollout restart deploy/n8n -n n8n
> ```
> 本番運用では、**必ず Ingress Controller (例: Nginx Ingress) と cert-manager を導入し、Let's Encrypt 等で TLS (HTTPS) 化してください。**  
> TLS 化が完了するまでは、上記の暫定対応で HTTP アクセスをお試しください。

---

## 2. 画面の構成
| 領域 | 説明 |
| --- | --- |
| **Canvas** | ワークフロー全体とノードをドラッグ＆ドロップで配置するエリア |
| **Node パネル** | 左側。HTTP / Slack / MySQL など利用可能なノード一覧 |
| **Execution Log** | 右側。実行履歴・デバッグログを確認 |
| **Settings** | 右上の歯車。グローバル設定、API キー、ユーザ管理 |

---

## 3. はじめてのワークフロー
> サンプル: Web API から JSON を取得し、Slack へ投稿

1. **HTTP Request** ノードを Canvas に配置  
   - Method: `GET`  
   - URL: `https://api.coindesk.com/v1/bpi/currentprice.json`
2. **Set** ノードを追加して `USD.rate` を取り出す (簡易的な例)
3. **Slack** ノードを追加  
   - Credential: 右側パネルから **New** を選択し、Bot Token (xoxb-...) を入力  
   - Channel: `#general`  
   - Text: `現在の Bitcoin 価格: {{$json["rate"]}} USD`
4. ノード間を接続し、画面右上 **▶︎ Execute Workflow** をクリック
5. Slack にメッセージが届けば成功

![sample](https://docs.n8n.io/images/nodes/slack/send-message.png)

---

## 4. スケジュール実行 (Cron)
1. **Cron** ノードをワークフローの最初に配置
2. `Mode: Custom Cron`、`Cron Expression: 0 * * * *` (毎正時)
3. HTTP → Slack の流れを接続
4. **Activate** ボタンを押すことで自動実行が有効化されます

---

## 5. Credentials (認証情報) の管理
- **Settings → Credentials** でサービス毎に再利用可能な認証情報を作成
- デフォルトでは暗号化キーがコンテナ再起動で失われる可能性があります。永続化したい場合は `N8N_ENCRYPTION_KEY` を `n8n-secret` に追加してください

```powershell
kubectl create secret generic n8n-secret \
  --from-literal=N8N_ENCRYPTION_KEY=$(openssl rand -hex 32) \
  -n n8n --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deploy/n8n -n n8n
```

---

## 6. ワークフローのエクスポート / インポート
| 操作 | 手順 |
| --- | --- |
| エクスポート (.json) | ワークフロー一覧 → 右の `⋯` → **Export** |
| インポート | `+` ボタン → **Import** → ファイル選択 |

Git と連携してバージョン管理する場合は [n8n/git-node](https://github.com/n8n-io/n8n/tree/master/packages/nodes-base/nodes/Git) ノードや CLI (`n8n import:workflow`) も利用できます。

---

## 7. エラー通知
1. **Error Trigger** ノードを作成（グローバルに 1 つ）
2. Slack や Email ノードを接続し、`{{$workflow.error.message}}` などのデータを送信

公式ガイド: <https://docs.n8n.io/hosting/logging-monitoring/error-handling/>

---

## 8. 参考リンク
- 公式ドキュメント: <https://docs.n8n.io/>  
- ノード別チュートリアル集: <https://n8n.io/integrations>  
- コミュニティ事例: <https://community.n8n.io/>  
- GitHub: <https://github.com/n8n-io/n8n>

---

以上で基本操作は完了です。ワークフローを構築しながら、不明点があればコミュニティや公式ドキュメントを参照してください。 