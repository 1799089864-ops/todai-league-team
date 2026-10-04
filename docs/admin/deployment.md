---
title: デプロイと運用
description: Cloudflare Workers へのデプロイ手順、更新方法、トラブルシューティング、ロールバックの注意点です。
order: 5
---

本ページは、システムを初めてデプロイする運営者や、更新作業を担当する運営者向けのガイドです。コマンドの詳細な動作については開発者に確認してください。

## 前提条件

作業を始める前に、以下を準備してください。

Git、Node.js、pnpmのインストールやリポジトリのクローンがまだの場合は、先に[ゼロからの環境構築](./getting-started.md)を済ませてください。

- [ ] Cloudflare アカウントを所持している
- [ ] ブラウザからアクセスできる（Deploy Button を使う場合）
- [ ] 本リポジトリをローカルに clone 済み（CLI を使う場合）
- [ ] Node.js 22 以上、pnpm がインストール済み（CLI を使う場合）
- [ ] `corepack enable && pnpm install` を実行済み（CLI を使う場合）

## ブラウザからの初回セットアップ（Deploy to Cloudflare ボタン）

ターミナルや Git 操作なしで、ブラウザから初回デプロイできます。

1. README 上部の **Deploy to Cloudflare** バッジをクリックします。
2. Cloudflare アカウントでログインし、表示されるプロンプトに従います。
3. Cloudflare がリポジトリを読み込み、新しい Worker と D1 データベース（`todai-league`）、Durable Objects（`LiveBoard`、`MatchActionCoordinator`）をプロビジョニングしてデプロイします。

この経路は **Deploy Button 専用の隔離された環境** を作成します。既存の `pnpm run deploy` 環境や手元の D1 データベースには接続しません。`wrangler.jsonc` の `d1_databases[0].database_id` は **省略したままである必要があり**（設定されている UUID がボタン生成のものであっても）、ボタンの `deploy:button` は固定データベース名 `todai-league` で migration を適用します。万一 `database_id` が記載されている構成では migration / deploy / secret 作成は一切行われず、ブラウザ専用のダッシュボード復旧手順を返します。

`BETTER_AUTH_SECRET` は `deploy:button` が Workers Builds 内で作成しようとします。成功すれば自動的に Worker が再デプロイされます。失敗した場合は、Cloudflare ダッシュボードから手動で作成し、再デプロイをトリガーしてください（下記「ボタン経路での復旧」を参照）。

- [Deploy to Cloudflare ボタン（公式）](https://developers.cloudflare.com/workers/tutorials/deploy-button/)
- [Workers Builds 概要](https://developers.cloudflare.com/workers/ci-cd/builds/)
- [Workers Builds 設定](https://developers.cloudflare.com/workers/ci-cd/builds/configuration/)

## 初回デプロイ（CLI 経由）

ターミナルが使える開発者や、staging 環境を含めて細かく制御したい場合はこちらを使ってください。ブラウザだけで済ませたい場合は上記の Deploy Button を利用してください。

### 1. Cloudflare へのログイン

ブラウザで [Cloudflare ダッシュボード](https://dash.cloudflare.com/) にログインするか、ターミナルで以下を実行してください。

```bash
pnpm exec wrangler login
```

ブラウザが開いて許可を求められたら、承認してください。

`pnpm run deploy` は環境チェック（preflight）を自動的に実行します。`pnpm preflight` はトラブルシューティング時に任意の診断コマンドとして個別に利用できます。

### 2. デプロイ実行

```bash
pnpm run deploy
```

`pnpm run deploy` は preflight を自動的に実行します。個別の `pnpm preflight` は任意の診断コマンドです。

`pnpm run deploy` は以下を順に行います。

1. 環境チェック (`pnpm preflight`)
2. アプリをビルドする
3. D1 binding の確認/プロビジョニング — `database_id` が未設定ならリモート DB の有無を確認し、存在しなければ作成して ID を wrangler 設定に書き戻す
4. 既存の D1 データベースに migration を適用する
5. Worker をデプロイする
6. `BETTER_AUTH_SECRET` が未設定なら自動生成して Cloudflare secret に登録する
7. 新しい secret を作成した場合のみ Worker を再デプロイする

`BETTER_AUTH_SECRET` は自動作成されます。Git に含めたり、チャットに貼ったりしないでください。

**初回デプロイの D1 作成**: リポジトリの wrangler 設定は初回デプロイ時まで `database_id` を省略しています。`pnpm run deploy` はこの状態を検出すると、`wrangler d1 list --json` でリモートに同名の D1 データベースが存在しないことを確認してから `wrangler d1 create` で作成します。作成後、`wrangler d1 info <name> --json` から取得した UUID を対象の wrangler 設定の `d1_databases[0].database_id` に書き込みます。**この設定ファイルの変更は必ずコミットしてください**。database_id は機密情報ではありません。

同名のデータベースがすでに存在するのに `database_id` が設定されていない場合、`pnpm run deploy` はエラーで停止します。復旧するには、既存のデータベース ID を以下のいずれかで確認し、対象の wrangler 設定に手動で追記してください。

- Cloudflare ダッシュボード: Workers & Pages → D1 → `<データベース名>` → database ID をコピー
- CLI: `pnpm exec wrangler d1 info <データベース名> --json` の `uuid` フィールド

手動で追記したら `pnpm run deploy` を再実行してください。

### 3. リソースの確認

初回セットアップは、ブラウザだけで済ませたい場合は README 上部の **Deploy to Cloudflare** バッジから、開発者が細かく制御したい場合は `pnpm run deploy` を使ってください。Deploy Button は **新規の隔離環境** の初回セットアップに使用できますが、既存の本番環境や staging 環境を更新する用途には使わず、Workers Builds を使用してください。**デプロイ後は必ず Cloudflare ダッシュボードで生成されたリソースを確認してください**。

確認すべき主なリソース：

- Workers & Pages（Worker 名・ルート・トリガー）
- D1 データベース（`todai-league` または `todai-league-staging`）
- Durable Objects（`LiveBoard`、`MatchActionCoordinator`）
- R2 / KV（バックアップ Worker を使う場合）

### 4. 管理者アカウントの作成

デプロイ後に表示された Worker URL を開き、以下にアクセスしてください。

```
https://<your-worker-url>/auth/bootstrap
```

`/auth/bootstrap` は初回のみ有効です。管理者アカウントを作成した後はアクセスできなくなります。

### 5. 大会の基本設定

管理者アカウントでログイン後、設定画面から以下を入力してください。

- 大会名
- 使用ルール
- その他の大会基本情報

### 6. ユーザー・チームの作成

- 「ユーザー管理」から運営アカウントや審判アカウントを発行する
- 「チーム」から参加チームと選手を登録する
- 必要に応じて A / B リーグに振り分ける

## 通常の更新：ターミナル不要（Workers Builds）

初回セットアップが完了し、以下が満たされている場合、以降のコード更新は **ブラウザだけ** で行えます。

- 本番用 D1 データベース（`todai-league`）が作成済み
- `wrangler.jsonc` の `d1_databases[0].database_id` にその UUID がコミット済み
- `BETTER_AUTH_SECRET` が Cloudflare ダッシュボードの **Secrets** から設定済み
- リポジトリが GitHub または GitLab に接続済み

### 操作手順

1. GitHub / GitLab の Web UI で main ブランチに変更をマージまたはプッシュします。
2. Cloudflare ダッシュボードの Workers Builds が自動的に以下を実行します。
   - Build command: `pnpm build`
   - Deploy command: `pnpm run deploy:workers-builds`

ダッシュボードから「Retry deploy」や手動トリガーを実行した場合も、同じ Build / Deploy コマンドが実行されます。

### ダッシュボード設定詳細（初回のみ）

1. [Cloudflare ダッシュボード](https://dash.cloudflare.com/) → Workers & Pages → 対象 Worker を開きます。
2. **Workers Builds** タブ（または CI/CD / Builds）から Git リポジトリを連携します。
3. 本番ブランチは `main` を指定します。
4. ビルド設定を入力します。

   | 項目           | 値                               |
   | -------------- | -------------------------------- |
   | Build command  | `pnpm build`                     |
   | Deploy command | `pnpm run deploy:workers-builds` |

5. Node.js 22 / pnpm は `package.json` の `packageManager` フィールドと corepack で自動的に有効化されます。ダッシュボードで Node バージョンを指定できる場合は `22` を選択してください。

### Workers Builds での secret 管理

`BETTER_AUTH_SECRET` は **必ず Cloudflare ダッシュボードの Secrets から作成** し、リポジトリやビルドログに含めないでください。`deploy:workers-builds` は秘密情報を一切生成・書き換えしません。

- [Worker Secrets 公式ドキュメント](https://developers.cloudflare.com/workers/configuration/secrets/)

### ロールバック

Cloudflare ダッシュボードから Worker のバージョンロールバックができます。ただし **D1 migration は自動で巻き戻りません**。データベースの変更を元に戻す必要がある場合は、開発者に相談してください。

### 公式ドキュメント

- [Workers Builds 概要](https://developers.cloudflare.com/workers/ci-cd/builds/)
- [Workers Builds 設定](https://developers.cloudflare.com/workers/ci-cd/builds/configuration/)
- [D1 Migrations](https://developers.cloudflare.com/d1/reference/migrations/)

## ターミナルからの本番更新

Workers Builds を使わず、開発者や特権を持つ運営者が直接実行する場合は `pnpm run deploy` を使います。

```bash
pnpm run deploy
```

`pnpm run deploy` は preflight を自動的に実行します。`pnpm preflight` は任意の診断コマンドとしてトラブルシューティング時にも個別に実行できます。

migration が追加されている場合は、Worker をデプロイする前に自動的に適用されます。通常の更新では Worker は 1 回のデプロイで済み、新しい secret を作成した場合のみ追加の再デプロイが行われます。

## Staging 環境

`wrangler.staging.jsonc` で staging Worker と D1（`todai-league-staging`）が定義されています。

```bash
pnpm run deploy:staging
```

staging は本番投入前の動作確認用です。本番データに影響を与えません。Staging も `scripts/deploy.mjs` でデプロイされ、本番と同じく D1 確認 → migration → deploy → secret の順序で実行されます。Staging は独立した D1 データベース（`todai-league-staging`）を使用し、初回デプロイ時に `wrangler.staging.jsonc` へその `database_id` が書き込まれます。

## ロールバック

Worker のコードと設定を直前のバージョンに戻すには、Cloudflare ダッシュボードから行うか、CLI を使います。

- ダッシュボード: Workers & Pages → 対象 Worker → **Deployments** → ロールバックしたいバージョンを選択
- CLI:

  ```bash
  pnpm exec wrangler rollback --config wrangler.jsonc
  ```

> [!WARNING]
> Worker のロールバックはコードと設定を巻き戻しますが、**D1 の migration は巻き戻しません**。データベースの変更を元に戻す必要がある場合は、開発者と相談してください。

## ボタン経路での復旧（ターミナル不要）

Deploy Button 経路で D1 migration や secret 作成に失敗した場合、ターミナルを使わずに Cloudflare ダッシュボードから復旧できます。

### D1 migration に失敗した場合

1. [Cloudflare ダッシュボード](https://dash.cloudflare.com/) → **Workers & Pages** → **D1** を開き、Deploy Button で作成された Worker 専用の `todai-league` データベースが存在するか確認します。プロビジョニングが完了していない場合は、ダッシュボードから作成・紐付けを行います。
2. `wrangler.jsonc` の `database_id` は **記載しないまま** にしておきます。Deploy Button 経路では既存 DB の誤アタッチを防ぐため、設定済みの ID は拒否されます。
3. バインディングの用意が確認できたら、ダッシュボードの Deploy Button / Workers Builds から **Retry deploy** を実行します。

### `BETTER_AUTH_SECRET` の作成に失敗した場合

1. ダッシュボード → **Workers & Pages** → 対象 Worker → **Secrets** を開きます。
2. `BETTER_AUTH_SECRET` を追加します（値は強固な乱数を安全な方法で生成してください）。
3. ダッシュボードから **Retry deploy** を実行します。

secret 値はリポジトリやビルドログに含めないでください。

## トラブルシューティング

### `pnpm preflight` で `[BLOCKER]` が出る

表示された項目を修正してください。多くの場合、以下のいずれかです。

- Node.js のバージョンが 22 未満
- `pnpm install` が未実行
- `wrangler login` が未実行
- `wrangler.jsonc` が見つからない

### `pnpm preview` で `BETTER_AUTH_SECRET is not configured`

`pnpm preview` は wrangler dev を使用するため、秘密鍵は `.dev.vars` から読み込まれます。`.dev.vars` を作成し、`BETTER_AUTH_SECRET` を設定してください。

```bash
pnpm dlx auth@latest secret
```

生成された値を `.dev.vars` に貼り付けます：

```
BETTER_AUTH_SECRET="<generated-secret>"
```

`.env` は drizzle-kit や Vite ビルド用の変数に使用するため、`pnpm preview` には反映されません。

### デプロイ中に secret の作成に失敗する

`wrangler` が Cloudflare に認証されていない可能性があります。`pnpm exec wrangler whoami` で確認し、必要に応じて `pnpm exec wrangler login` を実行してください。

### カスタムドメインを使いたい

`workers.dev` の URL でも動作します。カスタムドメインを強制したい場合のみ、以下を実行してください。

```bash
pnpm exec wrangler secret put BETTER_AUTH_URL --config wrangler.jsonc
```

値としてカスタムドメインの URL（例：`https://badminton.example.com`）を入力してください。

### Workers Builds で `d1_databases[0].database_id must be a valid UUID` と出る

`deploy:workers-builds` は既存のプロビジョニング済み環境専用です。`wrangler.jsonc` に本番 D1 の `database_id` が含まれていないとこのエラーになります。復旧手順：

1. Cloudflare ダッシュボード → D1 → `todai-league` → database ID をコピー
2. `wrangler.jsonc` の `d1_databases[0]` に `"database_id": "<コピーしたUUID>"` を追加
3. main ブランチへコミット・プッシュ

### Workers Builds で `BETTER_AUTH_SECRET is not configured` と出る

`BETTER_AUTH_SECRET` が Cloudflare ダッシュボードの secret として設定されていません。ダッシュボード → Workers & Pages → 対象 Worker → **Secrets** から `BETTER_AUTH_SECRET` を作成してください。値は `pnpm dlx auth@latest secret` などで生成できます。

## 開発者への引き継ぎ

以下は、運営者から開発者へ確認が必要なトピックです。

- migration の追加・変更履歴
- カスタムドメイン・`BETTER_AUTH_URL` の設定有無
- バックアップ Worker のデプロイ状況
- D1 / Durable Objects の手動補正が必要な事象
