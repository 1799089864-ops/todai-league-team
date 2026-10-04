# 東大リーグ団体戦 運営システム

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button.svg)](https://deploy.workers.cloudflare.com/?url=https://github.com/loveall-badminton/todai-league-team)

東大リーグ団体戦の運営とスコア管理を行うWebアプリです。大会の準備から試合の進行、結果の確定までをまとめてデジタルで扱えます。参加者、運営者、観戦者は、それぞれの立場に必要な情報を確認できます。

このシステムは東大リーグ団体戦に合わせた設計です。ほかの大会で使うこともできますが、個別の大会への対応や機能追加はお約束できません。

## 主な機能

- ライブスコア配信。各コートのスコアをリアルタイムに反映し、会場の外からも確認できます。
- オーダー管理。チームごとの種目エントリーを、提出、承認、公開の順に進めます。
- 対戦カード管理。グループステージと決勝トーナメントの日程、会場、進行状況を管理します。
- 順位の自動計算。勝敗と得失点差からリーグ順位を自動で集計します。

Git、Node.js、pnpmのインストールやリポジトリのクローンから始める場合は、[ゼロからの環境構築](./docs/admin/getting-started.md)を参照してください。何も入っていないパソコンから、ローカルでの起動、Cloudflareへのデプロイまでを順に説明しています。

## ブラウザから初回デプロイする（運営者向け）

冒頭のDeploy to Cloudflareバッジを使うと、ターミナルやGitを操作せずに初回デプロイできます。

1. バッジをクリックし、Cloudflareアカウントでログインします。あとは画面の指示に従ってください。
2. プロジェクト名などを確認して決めます。
3. Cloudflareがリポジトリを読み込みます。続けてD1データベース（`todai-league`）とDurable Objects（`LiveBoard`、`MatchActionCoordinator`）を新しく作成し、Workerをデプロイします。

この手順で作られるのは、Deploy Button専用の独立した環境です。既存の`pnpm run deploy`環境や手元のD1データベースには接続せず、ボタンが新しく作成したD1データベースを使います。

`BETTER_AUTH_SECRET`は、Workers Buildsの実行権限が足りていれば`deploy:button`が自動で作成し、Workerを再デプロイして反映します。作成に失敗した場合は、Cloudflareダッシュボードの「Workers & Pages」で対象のWorkerを開き、Secretsに`BETTER_AUTH_SECRET`を手動で登録してください。値には、安全な方法で生成した強い乱数を使います。登録したら、ダッシュボードの「Retry deploy」で再デプロイしてください。

> [!IMPORTANT]
> Deploy Button経由のデプロイでD1のmigrationやsecretの作成に失敗したときは、ターミナルのコマンドを使わず、Cloudflareダッシュボードから復旧してください。復旧手順は[docs/admin/deployment.md](./docs/admin/deployment.md)にあります。
> Deploy Buttonは、新しい独立環境を最初にセットアップするためのものです。既存環境の更新には[Workers Builds](#コードの更新をデプロイする運営者向けターミナル不要)を使ってください。

## ターミナルから初回デプロイする（開発者向け）

ターミナルを使える開発者や、staging環境も含めて細かく制御したい場合は`pnpm run deploy`を使います。ブラウザだけで済ませたい場合は、前節のDeploy Buttonを使ってください。

1. Cloudflareにログインします。ダッシュボードからでも、次のコマンドからでもかまいません。

   ```bash
   pnpm exec wrangler login
   ```

2. 依存パッケージをインストールします。

   ```bash
   pnpm install
   ```

3. 本番環境にデプロイします。

   ```bash
   pnpm run deploy
   ```

   `pnpm run deploy`は次の順に処理を進めます。preflightも自動で実行されるので、事前に`pnpm preflight`を実行する必要はありません。`pnpm preflight`は、トラブルが起きたときに単独で環境を診断するためのコマンドです。
   1. `pnpm preflight`で環境をチェックする
   2. `pnpm build`で型生成、svelte-check、Viteビルドを行う
   3. D1のbindingを確認する。`database_id`が未設定なら、リモートに同名のDBがあるかを確かめ、なければ作成してIDをwrangler設定に書き戻す
   4. `wrangler d1 migrations apply todai-league`でD1データベースにmigrationを適用する
   5. `wrangler deploy`でWorkerをアップロードする
   6. `scripts/ensure-cloudflare-secrets.mjs`を実行し、`BETTER_AUTH_SECRET`が未設定なら自動生成する
   7. 新しいsecretを作成した場合に限り、`wrangler deploy`でWorkerを再デプロイする

   CLIでデプロイすると`BETTER_AUTH_SECRET`は自動生成されます。Gitで管理したり、手動でコミットしたりする必要はありません。

   初回デプロイ時のD1作成について補足します。リポジトリのwrangler設定では、初回デプロイまで`database_id`を書いていません。`pnpm run deploy`はこの状態を検出すると、リモートに同名のD1データベースがないことを確かめてから`wrangler d1 create`で作成します。作成後は、Cloudflareから受け取ったUUIDを`wrangler.jsonc`（本番）または`wrangler.staging.jsonc`（staging）の`d1_databases[0].database_id`に書き込みます。**書き換わった設定ファイルは必ずコミットしてください。** database_idは機密情報ではありません。

   リモートに同名のデータベースがあるのに設定に`database_id`がない場合、`pnpm run deploy`はエラーを出して止まります。ダッシュボードか`pnpm exec wrangler d1 info todai-league --json`で既存データベースのIDを調べ、対象のwrangler設定に手で追記してから再実行してください。

4. デプロイ後に表示されたWorkerのURLを開き、`/auth/bootstrap`で管理者アカウントを作成します。`/auth/bootstrap`を使えるのは初回だけです。2回目以降は`/auth/login`からログインしてください。

5. 管理画面で、大会名やルールなどの基本設定を入力します。

6. ユーザーとチームを作成し、対戦を生成したら運用を始められます。

運用の詳しい手順は[docs/admin/deployment.md](./docs/admin/deployment.md)を参照してください。

## コードの更新をデプロイする（運営者向け、ターミナル不要）

初回セットアップが終わっていれば、以降のコード更新はブラウザだけでデプロイできます。前提として、本番D1の`database_id`が`wrangler.jsonc`にコミットされ、`BETTER_AUTH_SECRET`がCloudflareダッシュボードのsecretとして登録されている必要があります。

1. GitHubまたはGitLabのWeb画面で、変更をmainブランチにマージまたはプッシュします。
2. Cloudflare側でWorkers Buildsがビルドとデプロイを自動で実行します。使うコマンドは次の2つです。
   - Build command: `pnpm build`
   - Deploy command: `pnpm run deploy:workers-builds`

ダッシュボードの「Retry deploy」や手動トリガーを使った場合も、実行されるコマンドは同じです。

### Workers Buildsの初期設定チェックリスト

- [ ] リポジトリをGitHubまたはGitLabに置いている
- [ ] 本番のWorkerとD1データベース（`todai-league`）を作成済み
- [ ] 本番D1のUUIDを`wrangler.jsonc`の`d1_databases[0].database_id`に書き込み、コミット済み
- [ ] `BETTER_AUTH_SECRET`をCloudflareダッシュボードのSecretsに登録済み
- [ ] CloudflareダッシュボードでWorkers Buildsを有効にし、本番ブランチを`main`に設定済み
- [ ] Build commandを`pnpm build`に設定済み
- [ ] Deploy commandを`pnpm run deploy:workers-builds`に設定済み
- [ ] Node.js 22とpnpmを、corepackまたは`packageManager`フィールドで使える状態にしている

詳しくは[docs/admin/deployment.md](./docs/admin/deployment.md)と、次のCloudflare公式ドキュメントを参照してください。

- [Workers Builds 概要](https://developers.cloudflare.com/workers/ci-cd/builds/)
- [Workers Builds 設定](https://developers.cloudflare.com/workers/ci-cd/builds/configuration/)
- [D1 Migrations](https://developers.cloudflare.com/d1/reference/migrations/)
- [Worker Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)

> [!IMPORTANT]
> `deploy:workers-builds`は、作成済みのリソースに対してだけ動きます。D1の作成、database_idの書き込み、secretの生成は行わないので、これらは初回デプロイ時に`pnpm run deploy`かダッシュボードで済ませておいてください。

### ロールバック

WorkerのバージョンはCloudflareダッシュボードからロールバックできます。ただし、D1のmigrationは自動では巻き戻りません。データベースの変更を元に戻す必要があるときは、開発者に相談してください。

`BETTER_AUTH_SECRET`などの秘密情報は必ずダッシュボードのsecretで管理し、リポジトリやビルドログに含めないでください。

### ターミナルから更新する場合

Workers Buildsを使わずに開発者が直接デプロイするときは、初回と同じく`pnpm run deploy`を実行します。初回セットアップやstagingでの確認にも使うコマンドです。preflightは自動で実行されますが、トラブルシューティングのために`pnpm preflight`だけを単独で実行することもできます。

```bash
pnpm run deploy
```

## ローカルで開発する（開発者向け）

Git、Node.js 22以上、pnpmのインストール方法は[ゼロからの環境構築](./docs/admin/getting-started.md)にまとめています。

```bash
corepack enable
pnpm install
cp .env.example .env
```

`.env`には、drizzle-kitやViteのビルドで使う環境変数（`CLOUDFLARE_*`など）を書きます。`pnpm preview`のようにwrangler devで動かす場合は、wranglerが読み込む`.dev.vars`も作成し、`BETTER_AUTH_SECRET`を設定してください。値は次のコマンドで生成できます。

```bash
pnpm dlx auth@latest secret
```

生成した値を`.dev.vars`に貼り付けます。

```
BETTER_AUTH_SECRET="<generated-secret>"
```

UIだけを手早く確認するなら`pnpm dev`で十分です。

```bash
pnpm dev
```

D1、Durable Objects、認証、WebSocketまで含めて確認するなら`pnpm preview`を使います。`pnpm preview`はビルド結果を配信するので、先にビルドとローカルD1のテーブル作成を済ませてください。

```bash
pnpm build
pnpm db:reset:local
pnpm preview
```

起動したら`http://localhost:4173/auth/bootstrap`で管理者アカウントを作成します。コードを変更したときは、`pnpm build`をやり直してから再起動してください。

## よく使うコマンド

```bash
pnpm preflight                  # 環境チェック
pnpm run deploy                 # 本番デプロイ（deploy:prodと同じ。CLIやbootstrap用）
pnpm run deploy:staging         # stagingデプロイ
pnpm run deploy:workers-builds  # Workers BuildsのDeploy command用（既存環境のみ）
pnpm run deploy:button          # Deploy to Cloudflareボタンの初回デプロイ用（独立環境）
pnpm dev                        # Vite devサーバー（UIのみ）
pnpm preview                    # wrangler dev（D1、DO、認証あり）
pnpm build                      # 型生成、svelte-check、Viteビルド
pnpm check                      # gen + svelte-check
pnpm lint                       # Prettierのチェック + ESLint
pnpm format                     # Prettierで整形
pnpm test                       # vitest run
pnpm db:migrate:prod            # 本番D1にmigrationを適用
pnpm db:migrate:staging         # staging D1にmigrationを適用
```

## 注意事項

- `BETTER_AUTH_URL`の設定は任意です。未設定のときはリクエストのoriginを使うため、`workers.dev`のURLでも動きます。カスタムドメインに固定したい場合だけ、`wrangler secret put BETTER_AUTH_URL`で設定してください。
- バックアップ用のWorkerは必須ではありません。使う場合は`pnpm backup:deploy`で別にデプロイし、`BACKUP_WORKER_URL`と`BACKUP_DOWNLOAD_TOKEN`を`wrangler secret put`で設定してください。
- 大会当日に安定して運用するため、Cloudflare Workersの有料プランをおすすめします。利用料金は各主管団体が契約内容を確認し、自らの責任で負担してください。

## 開発とクレジット

- 開発: 東京大学ラブオール
- ソースコード: [GitHub](https://github.com/loveall-badminton/todai-league-team)

本システムは、東大リーグ団体戦の運営を支援する目的で東京大学ラブオールが開発したものです。

東京大学ラブオール以外の団体が主管する大会で本システムを使う場合、その大会の運用責任は主管団体が負うものとします。運用には、セットアップ、参加者情報の登録、試合進行、結果確定、公開情報の確認などが含まれます。

東京大学ラブオールは、本システムの開発や保守、必要に応じた技術的な助言を行うことがあります。ただし、ほかの団体が主管する大会について、運用上の判断、入力内容、公開情報、試合結果、そのほか本システムの利用で生じた不都合に対する責任は負いません。

## お問い合わせとフィードバック

バグ報告や機能の要望は[GitHub Issues](https://github.com/loveall-badminton/todai-league-team/issues)にお寄せください。機能追加やバグ修正のPull Requestも歓迎します。ただし、すべての提案や修正を採用するとは限りません。

## ライセンス

本システムはMIT Licenseで提供しています。詳しくは[LICENSE](./LICENSE)を参照してください。
