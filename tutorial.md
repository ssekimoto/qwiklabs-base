<walkthrough-metadata>
  <meta name="title" content="Cloud Run スターター ハンズオン">
  <meta name="description" content="Node.js と Express のアプリケーションをソースコードから Cloud Run へデプロイします。">
</walkthrough-metadata>

<walkthrough-disable-features toc>

# Cloud Run スターター ハンズオン

## はじめに

このハンズオンでは、HTTP リクエストに応答する小さな Node.js アプリケーションを作成し、Cloud Run にデプロイします。

Cloud Run は、コンテナ化されたアプリケーションを実行するフルマネージドのサーバーレス プラットフォームです。このハンズオンでは Dockerfile を自分で作成せず、`gcloud run deploy --source` と Google Cloud Buildpacks を利用して、ソースコードからコンテナイメージをビルドします。

### このハンズオンで学ぶこと

- Cloud Run、Cloud Build、Artifact Registry の API を有効にする
- Node.js / Express アプリケーションを準備する
- Cloud Shell 上でアプリケーションをローカル実行する
- ソースコードから Cloud Run へデプロイする
- サービス URL、リビジョン、ログを確認する
- 作成した Cloud Run サービスを削除する

### 作成する構成

```text
利用者
  |
  | HTTPS
  v
Cloud Run: starter-app
  |
  +-- Node.js 20
  +-- Express

ソースデプロイ時
  Cloud Shell -> Cloud Build -> Artifact Registry -> Cloud Run
```

### 所要時間の目安

約 15～25 分です。初回の API 有効化とソースビルドには数分かかることがあります。

### 前提条件

- 課金が有効な Google Cloud プロジェクト
- Cloud Shell を利用できること
- API 有効化、Cloud Run デプロイ、IAM 設定に必要な権限
- この Markdown ファイルを Cloud Shell の Teachme / Tutorial で開いていること


---

## Lab 00: Cloud Shell とプロジェクトを確認する

### 1. 作業ディレクトリを準備する

```bash
export HANDSON_HOME="$HOME/cloud-run-starter-lab"
mkdir -p "$HANDSON_HOME"
cd "$HANDSON_HOME"
pwd
```

次のように表示されれば準備完了です。

```text
/home/USER_NAME/cloud-run-starter-lab
```

### 2. プロジェクトと共通変数を設定する

```bash
export PROJECT_ID="${GOOGLE_CLOUD_PROJECT:-$(gcloud config get-value project 2>/dev/null)}"
export REGION="asia-northeast1"
export SERVICE_NAME="starter-app"
export APP_DIR="$HANDSON_HOME/work/starter-nodejs"

gcloud config set project "$PROJECT_ID"
gcloud config set run/region "$REGION"
gcloud config set run/platform managed
```

このハンズオンでは、Cloud Run のリージョンとして東京 `asia-northeast1` を使用します。

### 3. 設定値を確認する

```bash
printf 'PROJECT_ID   : %s\n' "$PROJECT_ID"
printf 'REGION       : %s\n' "$REGION"
printf 'SERVICE_NAME : %s\n' "$SERVICE_NAME"
printf 'APP_DIR      : %s\n' "$APP_DIR"

gcloud config list --format='text(core.project,run.region,run.platform)'
```

`PROJECT_ID` が空、または `(unset)` の場合は、Google Cloud コンソール上部で対象プロジェクトを選択してから Cloud Shell を再起動してください。

### 4. 実行環境を確認する

```bash
set -e

command -v gcloud >/dev/null
command -v node >/dev/null
command -v npm >/dev/null
command -v curl >/dev/null

printf 'Node.js : %s\n' "$(node --version)"
printf 'npm     : %s\n' "$(npm --version)"
printf 'gcloud  : %s\n' "$(gcloud version --format='value(Google Cloud SDK)' 2>/dev/null || gcloud version | head -n 1)"
printf 'project : %s\n' "$PROJECT_ID"
printf 'region  : %s\n' "$REGION"

echo "事前確認は完了しました。"
```

最後に `事前確認は完了しました。` と表示されれば成功です。

---

## Lab 01: 必要な API を有効にする

### 1. API を有効にする

```bash
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  logging.googleapis.com
```

このハンズオンでは次のサービスを利用します。

- **Cloud Run:** アプリケーションの実行
- **Cloud Build:** ソースコードからコンテナイメージをビルド
- **Artifact Registry:** ビルドしたコンテナイメージを保存
- **Cloud Logging:** アプリケーションログを保存・表示

### 2. API の状態を確認する

```bash
gcloud services list --enabled \
  --filter='config.name:(run.googleapis.com OR artifactregistry.googleapis.com OR cloudbuild.googleapis.com OR logging.googleapis.com)' \
  --format='table(config.name,title)'
```

4 つの API が表示されれば成功です。

### 3. Cloud Build 用サービスアカウントにビルド権限を付与する

現在の Cloud Run ソースデプロイでは、Cloud Build が使用する Compute Engine デフォルトサービスアカウントに Cloud Run Builder ロールが必要です。

```bash
export PROJECT_NUMBER="$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')"
export BUILD_SERVICE_ACCOUNT="${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:${BUILD_SERVICE_ACCOUNT}" \
  --role="roles/run.builder" \
  --condition=None
```

IAM の反映に少し時間がかかることがあります。権限付与を実行できない場合は、プロジェクト管理者に依頼してください。

### チェックポイント

`gcloud run deploy --source` は、Cloud Build と Buildpacks を使ってソースコードをコンテナイメージへ変換します。生成されたイメージは Artifact Registry の `cloud-run-source-deploy` リポジトリへ保存され、そのイメージを使って Cloud Run のリビジョンが作成されます。

---

## Lab 02: Node.js アプリケーションを準備する

### 1. 作業ディレクトリを作成する

```bash
rm -rf "$APP_DIR"
mkdir -p "$APP_DIR"
```

### 2. `package.json` を作成する

```bash
cd "$APP_DIR"

printf '%s\n' \
  '{' \
  '  "name": "cloud-run-starter-app",' \
  '  "version": "1.0.0",' \
  '  "private": true,' \
  '  "description": "Cloud Run starter application for a Teachme hands-on lab",' \
  '  "main": "index.js",' \
  '  "type": "module",' \
  '  "scripts": {' \
  '    "start": "node index.js"' \
  '  },' \
  '  "engines": {' \
  '    "node": "20.x"' \
  '  },' \
  '  "license": "Apache-2.0",' \
  '  "dependencies": {' \
  '    "express": "^5.2.1"' \
  '  }' \
  '}' \
  > package.json
```

### 3. `index.js` を作成する

```bash
printf '%s\n' \
  'import express from "express";' \
  '' \
  'const app = express();' \
  '' \
  'app.get("/", (req, res) => {' \
  '  console.log(JSON.stringify({' \
  '    severity: "INFO",' \
  '    message: "Received a request.",' \
  '    method: req.method,' \
  '    path: req.path,' \
  '  }));' \
  '' \
  '  res.status(200).type("text/plain").send("Hello Cloud Run!\n");' \
  '});' \
  '' \
  'const port = Number.parseInt(process.env.PORT ?? "8080", 10);' \
  '' \
  'app.listen(port, "0.0.0.0", () => {' \
  '  console.log(JSON.stringify({' \
  '    severity: "INFO",' \
  '    message: `Listening on port ${port}`,' \
  '  }));' \
  '});' \
  > index.js
```

### 4. `.gcloudignore` を作成する

```bash
printf '%s\n' \
  '.gcloudignore' \
  '.git' \
  '.gitignore' \
  'node_modules' \
  'npm-debug.log' \
  '.DS_Store' \
  > .gcloudignore
```

### 5. ファイル構成を確認する

```bash
find . -maxdepth 1 -type f -printf '%f\n' | sort
```

次の 3 ファイルが表示されます。

```text
.gcloudignore
index.js
package.json
```

### 6. `package.json` を確認する

```bash
sed -n '1,200p' package.json
```

主な設定は次のとおりです。

- `npm start` で `node index.js` を実行する
- Node.js 20 を使用する
- Web フレームワークとして Express を使用する

### 7. `index.js` を確認する

```bash
sed -n '1,240p' index.js
```

Cloud Run では、コンテナが環境変数 `PORT` で指定されたポートを待ち受ける必要があります。このアプリケーションは `PORT` が設定されていないローカル環境では 8080 を使用します。

### 8. 依存パッケージをインストールする

```bash
npm install
```

### 9. インストール結果を確認する

```bash
npm list --depth=0
node --version
ls -l package-lock.json
```

`express` と `package-lock.json` が確認できれば成功です。

---

## Lab 03: Cloud Shell 上でローカル実行する

### 1. アプリケーションを起動する

```bash
cd "$APP_DIR"
npm start > /tmp/cloud-run-starter-local.log 2>&1 &
export LOCAL_APP_PID=$!
sleep 3
```

### 2. HTTP リクエストを送信する

```bash
curl --fail --show-error --silent http://127.0.0.1:8080/
```

次のレスポンスが返れば成功です。

```text
Hello Cloud Run!
```

### 3. ローカルログを確認する

```bash
cat /tmp/cloud-run-starter-local.log
```

起動ログとリクエスト受信ログが JSON 形式で表示されます。

### 4. ローカルプロセスを停止する

```bash
kill "$LOCAL_APP_PID"
wait "$LOCAL_APP_PID" 2>/dev/null || true
```

### チェックポイント

Cloud Run にデプロイする前にローカルで HTTP 応答を確認することで、アプリケーションコードの問題とクラウド側の設定問題を切り分けやすくなります。

---

## Lab 04: Cloud Run へデプロイする

### 1. ソースコードからデプロイする

```bash
cd "$APP_DIR"

gcloud run deploy "$SERVICE_NAME" \
  --source . \
  --region "$REGION" \
  --allow-unauthenticated \
  --max-instances=3 \
  --quiet
```

このコマンドでは次の処理が自動的に行われます。

1. ソースコードが Cloud Build に送信される
2. Buildpacks が Node.js アプリケーションを検出する
3. コンテナイメージがビルドされる
4. Artifact Registry の `cloud-run-source-deploy` にイメージが保存される
5. Cloud Run の新しいサービスとリビジョンが作成される
6. 未認証アクセスが許可される
7. 最大インスタンス数が 3 に設定される

### 2. サービス URL を取得する

```bash
export SERVICE_URL="$(gcloud run services describe "$SERVICE_NAME" \
  --region "$REGION" \
  --format='value(status.url)')"

printf 'SERVICE_URL=%s\n' "$SERVICE_URL"
```

表示された URL をブラウザで開きます。

### 3. `curl` で動作確認する

```bash
curl --fail --show-error --silent "$SERVICE_URL"
```

次のレスポンスが返ればデプロイ成功です。

```text
Hello Cloud Run!
```

### 4. 複数回アクセスする

```bash
for request_number in 1 2 3 4 5; do
  printf 'request=%s response=' "$request_number"
  curl --fail --show-error --silent "$SERVICE_URL"
done
```

---

## Lab 05: Cloud Run のリソースとログを確認する

### 1. サービス情報を確認する

```bash
gcloud run services describe "$SERVICE_NAME" \
  --region "$REGION" \
  --format='yaml(metadata.name,status.url,status.latestReadyRevisionName,spec.template.metadata.annotations,spec.template.spec.containers)'
```

次の項目を確認してください。

- サービス名が `starter-app`
- URL が発行されている
- Ready 状態のリビジョンが存在する
- コンテナポートが 8080
- 最大インスタンス数が 3

### 2. リビジョン一覧を確認する

```bash
gcloud run revisions list \
  --service "$SERVICE_NAME" \
  --region "$REGION" \
  --format='table(metadata.name,status.conditions[0].status,status.conditions[0].type,metadata.creationTimestamp)'
```

Cloud Run のリビジョンは不変です。同じサービス名へ再デプロイすると、新しいリビジョンが作成されます。

### 3. アプリケーションログを確認する

```bash
gcloud run services logs read "$SERVICE_NAME" \
  --region "$REGION" \
  --limit=30
```

`Received a request.` と `Listening on port 8080` を含むログを確認します。

### 4. Cloud Build の履歴を確認する

```bash
gcloud builds list \
  --limit=5 \
  --format='table(id,status,createTime,duration,results.images)'
```

### 5. Artifact Registry リポジトリを確認する

```bash
gcloud artifacts repositories describe cloud-run-source-deploy \
  --location "$REGION" \
  --format='yaml(name,format,location,createTime)'
```

### チェックポイント

この時点で、アプリケーションコードだけでなく、ソースビルド、イメージ保管、Cloud Run リビジョン、ログまで一連の流れを確認できました。

Google Cloud コンソールでも次の画面を確認してください。

- [Cloud Run](https://console.cloud.google.com/run)
- [Cloud Build](https://console.cloud.google.com/cloud-build/builds)
- [Artifact Registry](https://console.cloud.google.com/artifacts)
- [Logs Explorer](https://console.cloud.google.com/logs/query)

---

## Lab 06: アプリケーションを変更して再デプロイする

Cloud Run では、同じサービス名へ再デプロイすると新しいリビジョンが作成されます。

### 1. 応答メッセージを変更する

```bash
cd "$APP_DIR"
sed -i "s/Hello Cloud Run!/Hello Cloud Run from Tokyo!/" index.js
grep -n "Hello Cloud Run" index.js
```

### 2. ローカルで再確認する

```bash
npm start > /tmp/cloud-run-starter-local-v2.log 2>&1 &
export LOCAL_APP_PID=$!
sleep 3
curl --fail --show-error --silent http://127.0.0.1:8080/
kill "$LOCAL_APP_PID"
wait "$LOCAL_APP_PID" 2>/dev/null || true
```

次のレスポンスが返ります。

```text
Hello Cloud Run from Tokyo!
```

### 3. 同じサービスへ再デプロイする

```bash
gcloud run deploy "$SERVICE_NAME" \
  --source . \
  --region "$REGION" \
  --allow-unauthenticated \
  --max-instances=3 \
  --quiet
```

### 4. 新しい応答を確認する

```bash
export SERVICE_URL="$(gcloud run services describe "$SERVICE_NAME" \
  --region "$REGION" \
  --format='value(status.url)')"

curl --fail --show-error --silent "$SERVICE_URL"
```

### 5. リビジョンが増えたことを確認する

```bash
gcloud run revisions list \
  --service "$SERVICE_NAME" \
  --region "$REGION" \
  --format='table(metadata.name,status.conditions[0].status,metadata.creationTimestamp)'
```

2 つ以上のリビジョンが表示されれば成功です。

---

## 完了

おめでとうございます。Node.js アプリケーションを Cloud Run にデプロイし、コード変更による新しいリビジョンの作成まで確認できました。

### 学習した内容

- Cloud Run ソースデプロイの構成
- Cloud Buildpacks による自動コンテナ化
- Artifact Registry へのイメージ保存
- Cloud Run サービスと不変リビジョン
- `PORT` 環境変数を使う HTTP サーバーの実装
- Cloud Logging と `gcloud` によるログ確認

---

## クリーンアップ

### 1. Cloud Run サービスを削除する

```bash
gcloud run services delete "$SERVICE_NAME" \
  --region "$REGION" \
  --quiet
```

### 2. 削除されたことを確認する

```bash
if gcloud run services describe "$SERVICE_NAME" --region "$REGION" >/dev/null 2>&1; then
  echo "Cloud Run サービスが残っています。"
else
  echo "Cloud Run サービスは削除されました。"
fi
```

### 3. Artifact Registry を削除する場合

`cloud-run-source-deploy` を他の Cloud Run ソースデプロイでも使用している場合は削除しないでください。このハンズオン専用プロジェクトであることを確認した場合のみ実行します。

```bash
gcloud artifacts repositories delete cloud-run-source-deploy \
  --location "$REGION" \
  --quiet
```

### 4. ローカル作業ファイルを削除する場合

```bash
rm -rf "$HANDSON_HOME/work"
rm -f /tmp/cloud-run-starter-local.log /tmp/cloud-run-starter-local-v2.log
```

---

## うまくいかない場合

### `PERMISSION_DENIED` または `roles/run.builder` が表示される

Cloud Run のソースデプロイは Cloud Build を利用します。Compute Engine デフォルトサービスアカウントに Cloud Run Builder ロールがない場合は、プロジェクト管理者が次を実行します。

```bash
export PROJECT_ID="${GOOGLE_CLOUD_PROJECT:-$(gcloud config get-value project)}"
export PROJECT_NUMBER="$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')"
export BUILD_SERVICE_ACCOUNT="${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:${BUILD_SERVICE_ACCOUNT}" \
  --role="roles/run.builder" \
  --condition=None
```

IAM の反映後、デプロイを再実行します。

### `--allow-unauthenticated` が組織ポリシーで拒否される

公開アクセスを許可できないプロジェクトでは、認証必須のままデプロイします。

```bash
cd "$APP_DIR"

gcloud run deploy "$SERVICE_NAME" \
  --source . \
  --region "$REGION" \
  --no-allow-unauthenticated \
  --max-instances=3 \
  --quiet
```

認証トークンを付けて呼び出します。

```bash
export SERVICE_URL="$(gcloud run services describe "$SERVICE_NAME" \
  --region "$REGION" \
  --format='value(status.url)')"

curl --fail --show-error --silent \
  -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  "$SERVICE_URL"
```

### ローカル確認でポート 8080 が使用中と表示される

```bash
lsof -iTCP:8080 -sTCP:LISTEN
```

不要なプロセスを停止した後、`npm start` を再実行します。

### デプロイに失敗したときのビルドログを確認する

```bash
gcloud builds list --limit=5
```

直近のビルド ID を取得し、詳細ログを表示します。

```bash
export BUILD_ID="$(gcloud builds list --limit=1 --format='value(id)')"
gcloud builds log "$BUILD_ID"
```

### Cloud Run のアプリケーションログを確認する

```bash
gcloud run services logs read "$SERVICE_NAME" \
  --region "$REGION" \
  --limit=50
```

### `PROJECT_ID` が空または `(unset)` になる

Google Cloud コンソール上部で対象プロジェクトを選択してから、Cloud Shell を再起動します。再起動後、次のコマンドで確認してください。

```bash
gcloud config get-value project
```

---

## 参考資料

- Google Codelabs: Cloud Run スターター チュートリアル
- Google Cloud: Deploy services from source code
- Google Cloud: Cloud Run quickstarts
