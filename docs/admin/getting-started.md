---
title: ゼロからの環境構築
description: Git・Node.js・pnpmのインストールからリポジトリのクローン、ローカルでの起動、Cloudflareへのデプロイまでを順に説明します。
order: 6
---

このページは、手元のパソコンにまだ何も入っていない状態から、本システムをローカルで動かし、Cloudflareにデプロイするまでの手順をまとめたものです。ターミナルを使うのが初めての運営者や、新しく開発に加わるメンバーを想定しています。

ブラウザだけで初回デプロイを済ませる場合は、このページの手順は必要ありません。READMEのDeploy to Cloudflareボタンを使ってください。

## 全体の流れ

作業は次の順に進めます。各章は前の章が終わっている前提で書いています。

1. GitHubとCloudflareのアカウントを用意する
2. ターミナルを使える状態にする
3. Gitをインストールする
4. Node.js 22以上をインストールする
5. pnpmを使える状態にする
6. リポジトリをクローンする
7. 依存パッケージをインストールし、環境をチェックする
8. ローカルで起動する
9. Cloudflareにデプロイする

ローカルで動かすだけなら8章まで、本番環境を作るなら9章まで進めてください。

## 1. アカウントを用意する

GitHubのアカウントは、ソースコードの取得と、変更を反映するときのプッシュに使います。[github.com](https://github.com/)で作成してください。

Cloudflareのアカウントは、本番環境へのデプロイに使います。[dash.cloudflare.com](https://dash.cloudflare.com/)で作成してください。ローカルで動かすだけなら、Cloudflareのアカウントは不要です。

大会当日に安定して運用するには、Cloudflare Workersの有料プランをおすすめします。料金は主管団体が契約内容を確認したうえで負担してください。

## 2. ターミナルを使える状態にする

以降の手順では、ターミナルにコマンドを入力して作業します。

macOSでは、Launchpadかアプリケーションフォルダの「ユーティリティ」から「ターミナル」を開いてください。

Windowsでは、WSL2上のUbuntuを使ってください。本システムのスクリプトは`rm -rf`やbashといったUnix系のコマンドを使っているため、PowerShellやコマンドプロンプトでは`pnpm build`などが失敗します。管理者権限のPowerShellで次のコマンドを実行し、再起動後にスタートメニューから「Ubuntu」を開きます。

```powershell
wsl --install
```

以降のコマンドは、すべてUbuntuのターミナルで実行してください。

Linuxでは、普段使っているターミナルをそのまま使えます。

## 3. Gitをインストールする

まず、Gitがすでに入っているかを確認します。

```bash
git --version
```

`git version 2.xx.x`のように表示されれば、この章は飛ばしてかまいません。表示されない場合は、OSごとに次の方法でインストールしてください。

macOSでは、次のコマンドを実行します。表示されるダイアログで「インストール」を選ぶと、Gitを含む開発ツール一式が入ります。

```bash
xcode-select --install
```

WSL2のUbuntuやLinuxで使うコマンドは次のとおりです。

```bash
sudo apt update
sudo apt install -y git
```

インストールが終わったら、コミットに記録する名前とメールアドレスを設定します。

```bash
git config --global user.name "あなたの名前"
git config --global user.email "you@example.com"
```

## 4. Node.js 22以上をインストールする

本システムはNode.js 22以上を前提にしています。CIはNode.js 22で動いており、Node.js 24でも動作を確認しています。

すでに入っているかを確認します。

```bash
node -v
```

`v22.x.x`以上が表示されれば、この章は飛ばしてかまいません。表示されない場合や、21以下のバージョンが表示された場合はインストールしてください。

いちばん手軽なのは、[Node.js公式サイト](https://nodejs.org/ja/download)からLTS版のインストーラーをダウンロードする方法です。macOSなら、pkgファイルを実行するだけで完了です。

複数のプロジェクトで異なるバージョンのNode.jsを使い分けるなら、バージョン管理ツールの利用をおすすめします。たとえば[mise](https://mise.jdx.dev/)を使う場合は、miseをインストールしたあとに次のコマンドを実行します。

```bash
mise use --global node@22
```

インストール後は、新しいターミナルを開いて`node -v`を再実行し、22以上が表示されることを確かめてください。

## 5. pnpmを使える状態にする

本システムのパッケージマネージャーはpnpmです。使うバージョンは`package.json`の`packageManager`フィールドで指定されています。Node.jsに付属するCorepackを有効にすると、プロジェクトのディレクトリでこのバージョンが自動で使われます。

```bash
corepack enable
```

Node.js 25以降にはCorepackが付属していません。`corepack: command not found`と表示された場合は、先にCorepackをインストールしてください。

```bash
npm install -g corepack
corepack enable
```

`corepack enable`で権限エラーが出る場合は、`sudo corepack enable`を試してください。

pnpmのバージョンは、次章でリポジトリをクローンしたあとに確認します。

## 6. リポジトリをクローンする

クローンとは、GitHub上のリポジトリを手元にコピーする操作です。

自分たちの団体で本番環境を運用する場合は、先にGitHub上で本リポジトリをフォークしてください。[リポジトリのページ](https://github.com/loveall-badminton/todai-league-team)右上の「Fork」を押すと、自分のアカウントか団体のOrganizationにコピーができます。Workers Buildsで自動デプロイするにはプッシュ権限が必要なので、フォークしたリポジトリをクローンします。

```bash
git clone https://github.com/<あなたのアカウント名>/todai-league-team.git
cd todai-league-team
```

ソースコードを読むだけ、ローカルで試すだけなら、本リポジトリを直接クローンしてもかまいません。

```bash
git clone https://github.com/loveall-badminton/todai-league-team.git
cd todai-league-team
```

以降のコマンドは、すべてこの`todai-league-team`ディレクトリの中で実行してください。

ここでpnpmのバージョンを確認します。

```bash
pnpm -v
```

`package.json`の`packageManager`と同じバージョンが表示されれば準備完了です。初回はCorepackがpnpmをダウンロードするので、確認を求められたら`Y`を入力してください。

GitHubへのプッシュでパスワードを求められたときは、[GitHub CLI](https://cli.github.com/)をインストールして`gh auth login`を実行しておくと、認証を任せられます。

## 7. 依存パッケージをインストールし、環境をチェックする

依存パッケージをインストールします。

```bash
pnpm install
```

インストールの最後には、Gitフック（lefthook）の設定も自動で行われます。以降は、コミット時にフォーマットや型、ユニットテストのチェックが走り、プッシュ時にE2Eテストが走ります。チェックが失敗した場合は、原因を直してからコミットやプッシュをやり直してください。`--no-verify`でフックを飛ばすことはしないでください。

次に、環境チェックを実行します。

```bash
pnpm preflight
```

各項目に`[OK]`、`[WARN]`、`[BLOCKER]`のどれかが表示されます。この時点では、次の2項目が`[OK]`以外になっていても問題ありません。

- `Cloudflare authentication`はまだログインしていないため`[BLOCKER]`になります。9章で解消します。
- `Local Better Auth secret present`は`.dev.vars`をまだ作っていないため`[WARN]`になります。8章で解消します。

それ以外の項目が`[BLOCKER]`になっている場合は、該当する章に戻って確認してください。

## 8. ローカルで起動する

ローカルでの起動方法は2つあります。画面の見た目だけを確認するなら`pnpm dev`、データベースやログインまで含めて動かすなら`pnpm preview`を使います。

### 画面だけを確認する

```bash
pnpm dev
```

ブラウザで`http://localhost:5173`を開いてください。この方法ではD1データベース、Durable Objects、認証が動かないため、ログインやスコア入力は試せません。終了するにはターミナルで`Ctrl + C`を押します。

### データベースやログインも含めて動かす

`pnpm preview`は、本番に近い構成でアプリを起動します。初回は次の手順で準備してください。

まず、環境変数ファイル`.env`を作成します。値は空のままでかまいません。リモートのD1をdrizzle-kitで操作するときにだけ値が必要になります。

```bash
cp .env.example .env
```

次に、認証用の秘密鍵を生成します。

```bash
pnpm dlx auth@latest secret
```

プロジェクト直下に`.dev.vars`というファイルを作り、表示された値を次の形式で書き込んでください。`.dev.vars`はGitの管理対象外なので、コミットされません。

```
BETTER_AUTH_SECRET="<生成された値>"
```

続けて、アプリをビルドし、ローカルのD1データベースにテーブルを作成します。

```bash
pnpm build
pnpm db:reset:local
```

`pnpm preview`はビルド結果を配信するため、ビルドしないまま起動すると`The directory specified by the "assets.directory" field ... does not exist`というエラーで止まります。

準備ができたら起動してください。

```bash
pnpm preview
```

ブラウザで`http://localhost:4173/auth/bootstrap`を開き、管理者アカウントを作成してください。以降は`http://localhost:4173/auth/login`からログインできます。

### サンプルデータを入れる

大会の途中を再現したサンプルデータで画面を確認したい場合は、`pnpm preview`を起動したまま、別のターミナルで次のコマンドを実行します。

```bash
pnpm db:seed
```

投入されるのは、12チーム分の対戦やオーダー、試合データです。ログインには次のアカウントを使えます。

| 種類       | ID                   | パスワード       |
| ---------- | -------------------- | ---------------- |
| 管理者     | `testadmin`          | `TestAdmin123`   |
| チーム     | `team01`から`team12` | `TeamPass123`    |
| 一般参加者 | `participant01`      | `Participant123` |

> [!CAUTION]
> `pnpm db:seed:staging`と`pnpm db:seed:prod`は、リモートのstaging環境や本番環境にデータを書き込みます。ローカルで試すときは、必ず`pnpm db:seed`を使ってください。

### コードを変更したとき

`pnpm preview`はビルド結果を配信しているので、ソースコードを変えても自動では反映されません。`pnpm preview`を`Ctrl + C`で止め、`pnpm build`を実行してから再び起動してください。画面の調整だけなら、変更が即座に反映される`pnpm dev`のほうが手早く確認できます。

ローカルのデータをすべて消して最初からやり直すときは、`pnpm db:reset:local`を実行してください。

## 9. Cloudflareにデプロイする

ターミナルからCloudflareにログインします。ブラウザが開いて許可を求められるので、承認してください。

```bash
pnpm exec wrangler login
```

`pnpm preflight`を再実行し、`[BLOCKER]`がなくなったことを確かめます。

```bash
pnpm preflight
```

本番環境にデプロイします。

```bash
pnpm run deploy
```

初回は、D1データベースの作成、テーブルの作成、Workerのアップロード、`BETTER_AUTH_SECRET`の生成までがすべて自動です。処理の詳しい順番は[デプロイと運用](./deployment.md)を参照してください。

初回デプロイでは、作成したD1データベースのIDが`wrangler.jsonc`に書き込まれます。この変更は必ずコミットしてプッシュしてください。

```bash
git add wrangler.jsonc
git commit -m "Add production D1 database_id"
git push
```

デプロイが終わると、ターミナルにWorkerのURLが表示されます。そのURLの`/auth/bootstrap`を開き、本番環境の管理者アカウントを作成してください。

```
https://<表示されたWorkerのURL>/auth/bootstrap
```

管理者アカウントを作成したら、[大会のセットアップ](./setup.md)に進みます。

2回目以降のコード更新を、ターミナルを使わずにGitHubへのプッシュだけでデプロイしたい場合は、Workers Buildsを設定してください。設定方法は[デプロイと運用](./deployment.md)の「通常の更新」にあります。

## 最新のコードに更新する

本リポジトリの更新を手元に取り込むときは、次の順に実行してください。

```bash
git pull
pnpm install
pnpm build
```

フォークしたリポジトリを使っている場合は、先にGitHub上のフォークのページで「Sync fork」を押し、本リポジトリの変更をフォークに取り込んでから`git pull`してください。

更新でデータベースの構造が変わっていることがあります。ローカルのデータを残したままテーブルの変更だけを反映するには、次のコマンドを実行します。

```bash
pnpm exec wrangler d1 migrations apply todai-league --local
```

## 開発者向け: テストを実行する

ユニットテストの一部はブラウザ上で動くため、初回だけPlaywright用のChromiumをインストールしてください。

```bash
pnpm exec playwright install chromium
```

そのうえで、次のコマンドでテストを実行できます。

```bash
pnpm test      # ユニットテストとコンポーネントテスト
pnpm test:e2e  # E2Eテスト（サーバーは自動で起動します）
pnpm lint      # フォーマットとESLintのチェック
pnpm check     # 型チェック
```

コードの構成や設計方針は、リポジトリ直下の`AGENTS.md`にまとめています。

## よくあるトラブル

### `node: command not found`や`pnpm: command not found`と表示される

インストール直後のターミナルでは、新しく入ったコマンドがまだ認識されていない場合があります。ターミナルを開き直してから再実行してください。それでも表示される場合は、4章と5章をやり直してください。

### `'rm' は、内部コマンドまたは外部コマンド…として認識されていません`と表示される

WindowsのPowerShellやコマンドプロンプトで実行しています。2章の手順でWSL2のUbuntuを用意し、Ubuntuのターミナルで作業してください。

### `pnpm preview`で`assets.directory`のエラーが出る

ビルド結果がありません。`pnpm build`を実行してから`pnpm preview`を起動してください。

### `pnpm preview`で`BETTER_AUTH_SECRET is not configured`と表示される

`.dev.vars`がないか、`BETTER_AUTH_SECRET`の行が書かれていません。8章の手順で作成してください。`.env`に書いても`pnpm preview`には反映されません。

### `/auth/bootstrap`を開くと500エラーになる

ローカルのD1データベースにテーブルがない可能性があります。`pnpm preview`のターミナルに`Failed query`というエラーが出ていれば、これが原因です。`pnpm preview`を止めて`pnpm db:reset:local`を実行し、再び起動してください。

### `Address already in use`と表示される

別の`pnpm preview`や`pnpm dev`が起動したままになっています。そのターミナルで`Ctrl + C`を押して止めてから、再び起動してください。

デプロイに関するトラブルは、[デプロイと運用](./deployment.md)の「トラブルシューティング」も参照してください。
