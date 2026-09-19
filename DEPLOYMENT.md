# Cinera デプロイガイド

このドキュメントでは、CineraをRender、Railway、Docker対応サービス、Vercel、Netlifyへデプロイする方法を説明します。

## 1. 先に確認すること

CineraはNode.jsのHTTPサーバーです。以下の機能を使用します。

- Node.js 20以上
- ホストが割り当てる`PORT`環境変数
- HLS中継のための長時間HTTPレスポンス
- 「FFMPEGで再生」と完全保存ダウンロードで使用するffmpeg
- HLS中継URLの署名に使用する`SESSION_SECRET`

### サービスの選び方

| サービス | 推奨用途 | ffmpeg | HLS中継 | 設定ファイル |
| --- | --- | --- | --- | --- |
| Render | 全機能を使う本番環境 | Docker内で利用可能 | 利用可能 | `render.yaml` |
| Railway | 全機能を使う本番環境 | Docker内で利用可能 | 利用可能 | `railway.json` |
| Docker対応サービス | 自分で実行環境を管理 | `Dockerfile`で導入 | 利用可能 | `Dockerfile` |
| Vercel | サーバーレスでの画面・API確認 | 利用不可 | 実行時間制限あり | `vercel.json` |
| Netlify | Functionsでの画面・API確認 | 利用不可 | 実行時間制限あり | `netlify.toml` |

FFMPEGリマックスと完全保存ダウンロードを含む全機能が必要な場合は、Render、Railway、またはDocker対応サービスを使用してください。

## 2. 共通の環境変数

### `SESSION_SECRET`

本番環境では、必ずホスティングサービスのSecret / Environment Variables機能で設定してください。

```text
SESSION_SECRET=十分に長いランダムな文字列
```

`SESSION_SECRET`を設定しない場合、プロセス起動時にランダム値が生成されます。再起動すると既存の署名付きHLS URLが無効になるため、本番では固定値を設定してください。

### `PORT`

アプリは`PORT`を自動的に読み取ります。Render、Railway、Vercel、Netlifyが自動で設定する場合は手動設定不要です。

ローカルまたはDockerで指定する場合:

```bash
PORT=5000 npm start
```

## 3. Render

Renderでは、ffmpegを含むDockerイメージを使用します。

### 手順

1. RenderでGitHubリポジトリを接続する
2. Blueprint / Infrastructure as Codeとしてリポジトリを選択する
3. `render.yaml`を使ってサービスを作成する
4. `SESSION_SECRET`が生成されていることを確認する
5. デプロイ後、サービスURLの`/`を開く

リポジトリに含まれている`render.yaml`は、次の設定を行います。

- Dockerランタイムを使用
- `Dockerfile`をビルド
- `/`をヘルスチェック
- `SESSION_SECRET`を自動生成

Renderのサービス設定で`PORT`を固定値にする必要はありません。

## 4. Railway

Railwayでも同じDockerfileを使用するため、ffmpegを含む全機能を実行できます。

### 手順

1. RailwayでGitHubリポジトリを追加する
2. リポジトリのサービスを作成する
3. `railway.json`と`Dockerfile`が検出されていることを確認する
4. Variablesに`SESSION_SECRET`を追加する
5. デプロイ後、発行されたURLの`/`を開く

`railway.json`では、Dockerfileビルド、`npm start`、`/`のヘルスチェック、失敗時の再起動を設定しています。

## 5. Docker対応サービス

Render・Railway以外のDocker対応サービスでも、プロジェクトの`Dockerfile`を使用できます。

### ローカルでビルド

```bash
docker build -t cinera .
```

### ローカルで起動

```bash
docker run --rm \
  -p 5000:5000 \
  -e SESSION_SECRET="change-this-to-a-long-random-value" \
  cinera
```

ブラウザで`http://localhost:5000`を開きます。

Dockerfileでは以下を行っています。

- Node.js 20を使用
- Debianのffmpegをインストール
- `npm ci --omit=dev`で依存関係をインストール
- `node`ユーザーでアプリを実行
- ポート5000を公開

ホスティングサービスが`PORT`を別の値に設定する場合、アプリ側はその値を自動的に使用します。Dockerのポート公開設定はサービスの仕様に合わせてください。

## 6. Vercel

Vercel用に次のファイルを含めています。

- `vercel.json`
- `api/index.js`

GitHubリポジトリをVercelにImportすると、`api/index.js`がNode.js Functionとして使われます。全ルートはこのFunctionへ書き換えられ、既存のアプリルーターがリクエストを処理します。

### 手順

1. VercelでGitHubリポジトリをImportする
2. Framework Presetは、必要に応じてOtherを選択する
3. Environment Variablesに`SESSION_SECRET`を追加する
4. デプロイする
5. 発行されたURLの`/`を開く

### Vercelでの制約

- VercelのFunction環境にffmpegを前提としてインストールできない
- 「FFMPEGで再生」は使用できない
- 完全保存ダウンロードは使用できない
- HLS中継はFunctionの実行時間・レスポンスストリーミング制限の影響を受ける
- 長時間の動画再生や大きなレスポンスはDocker対応サービスの方が安定する

画面表示や短時間のAPI確認を目的に使い、全機能の本番運用にはRender、Railway、またはDocker対応サービスを使用してください。

## 7. Netlify

Netlify用に次のファイルを含めています。

- `netlify.toml`
- `netlify/functions/server.js`
- `serverless-http`

`netlify/functions/server.js`が既存のNode HTTPサーバーをNetlify Functionとして呼び出します。`netlify.toml`のcatch-all redirectによって、画面とAPIのリクエストをFunctionへ送ります。

### 手順

1. NetlifyでGitHubリポジトリをImportする
2. Build commandが`npm ci`になっていることを確認する
3. Functions directoryが`netlify/functions`になっていることを確認する
4. Environment variablesに`SESSION_SECRET`を追加する
5. デプロイする
6. 発行されたURLの`/`を開く

### Netlifyでの制約

- Netlify Functionsにffmpegを前提としてインストールできない
- FFMPEGリマックスと完全保存ダウンロードは使用できない
- HLS中継はFunctionsの実行時間・レスポンスサイズ・ストリーミング制限の影響を受ける
- 長時間の動画配信にはDocker対応サービスを推奨する

## 8. デプロイ後の確認

まずトップページが表示されることを確認します。

```bash
curl -I https://YOUR_DOMAIN/
```

次に、ブラウザから以下を確認します。

1. 検索画面が表示される
2. 作品検索が実行できる
3. 作品詳細が表示される
4. 通常のHLS再生が開始できる
5. Docker対応サービスではFFMPEGリマックスが動作する
6. Docker対応サービスではダウンロード処理が動作する

## 9. トラブルシュート

### `Application failed to respond`

- サービスの起動コマンドが`npm start`になっているか確認する
- サーバーが`0.0.0.0`で待ち受けているか確認する
- `PORT`をアプリ側で固定していないか確認する
- Dockerサービスではコンテナログを確認する

### `ffmpeg: not found`

VercelまたはNetlifyで発生する場合は仕様上の制約です。Render、Railway、またはDocker対応サービスへ移行してください。

### `SESSION_SECRET`に関するエラー

- 環境変数名が正確に`SESSION_SECRET`になっているか確認する
- SecretをPreview環境とProduction環境の両方に設定する
- 再起動後に古い署名付きURLを再利用していないか確認する

### 外部APIは動くがHLS再生できない

- 配信元URLの有効期限が切れていないか確認する
- 配信元がホスティングサービスのIPやRefererを拒否していないか確認する
- サーバーレス環境では実行時間制限に達していないか確認する
- 全機能が必要な場合はDocker対応サービスで再確認する

## 10. デプロイ前チェックリスト

- [ ] `SESSION_SECRET`をSecretとして設定した
- [ ] 本番サービスに適した設定ファイルを選んだ
- [ ] 全機能が必要ならDocker対応サービスを選んだ
- [ ] `npm ci`が外部環境で成功する
- [ ] `/`のヘルスチェックが成功する
- [ ] 外部APIと配信元の利用規約・権利関係を確認した
- [ ] ログにSecretや署名付きURLを出していない