# HOMEWARD — 自分のCloudflareで使う版

GitHubで管理し、自分のCloudflare Workersに公開する夫婦用Webアプリです。
OpenAI SitesやChatGPTログインは必要ありません。GitHub Pages単体では動きません。

## アプリの機能

- 大きな「今から帰る」ボタン、任意のひと言、相手への本文付きWeb Push通知
- 文字・写真チャット（10秒ごとに更新）
- 月表示カレンダー、予定の追加・編集・削除
- 日付マス内に時刻と予定名を表示
- メールアドレス・パスワードでの専用ログイン、二人限定の招待登録
- iPhoneホーム画面用アイコン、通知登録、テスト通知

## 何をどこに置くか

- GitHub：プログラム
- 自分のCloudflare Workers：画面とAPI、通知送信
- 自分のCloudflare D1：アカウント、予定、チャット、縮小写真、通知登録と通知用の鍵

写真は端末側で長辺1280px以下のJPEGへ縮小して保存します。元写真の保管サービスではありません。
写真保存もD1で行うため、この版ではR2の契約・設定は不要です。
以前のSites版のデータは自動では移りません。新しいURLでは二人とも通知の登録をやり直します。

## Cloudflare管理画面から公開する手順

1. Workers & Pages → Create application → Continue with GitHub
2. `yohaku1228/huhu` リポジトリを選択
3. Project name：`huhu`
4. Build command：`npm run build`
5. **Deploy command：`npm run deploy`**（標準の `npx wrangler deploy` から変更）
6. 初期設定が終わるまでは Enable Preview builds をOFF
7. Deployを実行

`npm run deploy` は、Workersの公開・D1の自動作成・DBの初期化を行います。
Cloudflare側の権限や自動作成機能が有効な状態である必要があります。
実際の自分のアカウントへのデプロイは、接続・認可後に確認してください。
DB初期化中は一時的に準備中と表示されます。デプロイが失敗した場合はログを確認してください。

### 最初の登録を保護する設定

公開後、Cloudflareで `huhu` を開き、Settings → Variables and Secrets から以下を追加します。

| 種類 | 名前 | 値 |
| --- | --- | --- |
| Secret | `SETUP_KEY` | 自分で決めた24文字以上の長いランダムな文字列 |

秘密キーはGitHubやチャットに貼らないでください。
キーの変更を適用するため、Cloudflareの画面に従ってDeploy/Saveします。
公開したURLを開き、メールアドレス・表示名・12文字以上のパスワード・SETUP_KEYを入力して最初の登録をします。
SETUP_KEYが未設定の場合は、誰も初回登録できません。
初回登録後は設定から招待リンクをコピーし、奥さんに送ります。奥さんは招待リンクから登録します。
二人が登録すると、その招待リンクでは追加登録できません。
メールアドレスはログイン用の識別子です。認証メールや通知メールを送る機能はありません。

SETUP_KEYはパスワード再設定にも使います。安全な場所に保管してください。
紛失した場合はCloudflareで新しい値に変更して適用できます。変更だけで既存アカウントのパスワードは変わりません。

### 通知

通知に必要なVAPID鍵はアプリが生成し、D1のサーバー側だけで保持します。
秘密鍵を画面やAPIから返すことはありません。D1全体のエクスポートには秘密情報も含まれるので、GitHubに置かないでください。

1. 二人ともSafariで公開URLを開く
2. 「ホーム画面に追加」し、そのアイコンから開く
3. ログイン
4. 「通知の設定」→「この端末の通知を有効にする」
5. 「自分にテスト通知を送る」で着信を確認
6. 片方の「今から帰る」で、もう片方に着信するか確認

iOS 16.4以降が必要です。通信や集中モード、通知プレビュー等の設定によって表示・タイミングは変わります。
送信サービスの受付は、端末への着信や既読の確認ではありません。
通知の画像添付は行わず、写真は通知をタップしてチャットから開きます。

## ローカルで開発する

Node.js 22.13以上とnpmを用意します。

```sh
npm ci
```

`.dev.vars.example` を `.dev.vars` にコピーし、開発用のSETUP_KEYを24文字以上で入力します。
本番と同じキーは使わないでください。`.dev.vars` はGitHubにアップロードしません。

```sh
npm run db:local
npm run dev
```

`http://localhost:5174/` を開きます。ローカルでも本物と同じ専用ログインを使います。

```sh
npm run check
npm run build
npx wrangler deploy --dry-run
```

### テスト

`tests/smoke.mjs` は空の使い捨てローカルDBと起動中の開発サーバーに対して実行します。
本番URLでは動かないよう制限しています。
TEST_SETUP_KEY環境変数に開発用キーを設定してから `node tests/smoke.mjs` を実行してください。
テスト用アカウント、チャット、写真がローカルに作られます。
`tests/push-runtime.mjs` はそのローカルテスト環境向けです。実在しないテスト送信先の失効応答で、Cloudflareランタイムの暗号化・送信処理を検証します。

確認済み：本番ビルド、型チェック、ローカルの二人登録、招待制限、ログイン、パスワード再設定、セッション無効化、予定CRUD、帰宅連絡共有、写真アクセス制御、通知用公開鍵の保持、ヘッダー偽装拒否、Originチェック、CloudflareランタイムでのWeb Push暗号化・署名・送信と失効処理。
**未確認：あなたのCloudflareでの本番デプロイとiPhone実機への通知着信。**

## 更新・データ・運用

GitHubを更新すると、接続したCloudflare側でビルドと公開が実行される構成です。
Worker名はCloudflareのProject nameと `wrangler.jsonc` のnameを一致させます。
適用済みSQLは書き換えず、`drizzle/` に次の番号のSQLを追加してください。
`npm run deploy` が未適用のSQLのみ反映します。バックアップはCloudflareのD1管理機能から別途保管してください。
公開サイトを削除してもDBが残る場合があるため、廃止時はWorkersとD1をそれぞれ確認します。
プランの利用制限や料金はCloudflareの管理画面で確認してください。このソースは無料利用を保証するものではありません。
GitHubリポジトリが公開でも、ログイン情報や家族のデータはそのリポジトリに含まれません。

## ファイル

- `app/homeward.tsx`：帰宅ボタン、チャット、カレンダー
- `app/login.tsx`：ログインと初回登録画面
- `app/auth.ts`：パスワードハッシュとセッション
- `app/api/auth/route.ts`：招待登録、ログイン、ログアウト、パスワード再設定
- `app/api/family/route.ts`：家族の共有データ
- `app/api/photo/route.ts`：ログインした家族だけの写真
- `app/api/push/route.ts`、`app/push.ts`、`public/sw.js`：通知
- `wrangler.jsonc`、`vite.config.ts`、`build/worker.ts`：自分のCloudflare向け構成
- `scripts/deploy.mjs`：公開とDB初期化

以前のテンプレートの未使用コンポーネント類は残しています。同梱LICENSE等と各依存パッケージのライセンスも確認してください。
