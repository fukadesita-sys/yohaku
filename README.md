# 余白｜YOHaku

スマホ・PC対応のライフログWebアプリです。日々の記録は初期状態では各ブラウザ内に保存されます。複数端末で共有するには、HTTPSで公開し、Supabaseのクラウド同期を設定してください。

## 1. Web公開（GitHub Pages）

1. GitHubにログインし、新しいリポジトリを作成します（例：`yohaku`）。公開リポジトリには個人的な記録やバックアップを絶対に入れないでください。このフォルダのアプリファイルだけを置きます。
2. `index.html`、`manifest.json`、`icon.svg`、`sw.js`、`README.md` をリポジトリの直下（root）にアップロードしてコミットします。
3. リポジトリの **Settings → Pages** を開き、Build and deployment を **Deploy from a branch**、Branch を **main**、Folder を **/(root)** にして保存します。
4. GitHub Pagesが公開URLを表示するまで待ちます。URLは通常 `https://<GitHubユーザー名>.github.io/<リポジトリ名>/` の形式です。
5. 公開URLをスマホとPCの両方で開きます。AndroidのChromeではメニューから「ホーム画面に追加」または「アプリをインストール」を選べます。

アプリ本体は静的ファイルです。アプリの記録データ、JSONバックアップ、パスワード、Supabaseのsecret/service_role keyをGitHubにアップロードしないでください。GitHub Pagesは公開Webサイトなので、個人の記録をリポジトリへ保存しないこと。

## 2. Supabaseプロジェクトを作る

1. Supabaseで新しいプロジェクトを作成し、データベースの準備が終わるまで待ちます。
2. プロジェクトの **Connect / API Keys / Project Settings → API** などの画面で、Project URL と公開用キー（`anon` または `publishable`）を確認します。画面名称は変更される場合があります。
3. **SQL Editor** で、下記SQLを一度実行します。
4. Supabaseの **Authentication → URL Configuration** で、Site URL と Redirect URLs にGitHub Pagesの公開URL（例 `https://<GitHubユーザー名>.github.io/<リポジトリ名>/`）を設定します。メール確認が有効なら、アカウント作成後に確認メールを開いてからアプリでログインしてください。メール確認を無効にするかどうかは、自分のセキュリティ要件に合わせて決めてください。

### SQL（ユーザー自身の行だけアクセス可能）

```sql
create table if not exists public.yohaku_sync (
  user_id uuid primary key references auth.users(id) on delete cascade,
  payload jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now()
);

alter table public.yohaku_sync enable row level security;

drop policy if exists "Users can read own YOHaku data" on public.yohaku_sync;
drop policy if exists "Users can insert own YOHaku data" on public.yohaku_sync;
drop policy if exists "Users can update own YOHaku data" on public.yohaku_sync;

create policy "Users can read own YOHaku data"
on public.yohaku_sync for select
to authenticated
using (auth.uid() = user_id);

create policy "Users can insert own YOHaku data"
on public.yohaku_sync for insert
to authenticated
with check (auth.uid() = user_id);

create policy "Users can update own YOHaku data"
on public.yohaku_sync for update
to authenticated
using (auth.uid() = user_id)
with check (auth.uid() = user_id);
```

**絶対にアプリへ入力しないもの：** `service_role` key、secret key、データベースパスワード。アプリに入力するのは Project URL と公開用キーだけです。公開用キーは単独でデータへアクセスできないよう、RLSを必ず有効にします。

## 3. アプリをクラウドへ接続する

1. 公開URLを開き、「設定」→「クラウド同期」へ進みます。
2. Project URL と公開用キーを入力し、「接続設定を保存」します。
3. メールアドレスとパスワードを入力して「アカウント作成」を押します。
4. 確認メールが届いた場合はリンクを開き、アプリに戻って同じメールアドレス・パスワードで「ログイン」します。
5. 既存の記録がある端末では、先に「バックアップを書き出す」を押し、「今すぐ同期」を実行します。
6. 別の端末でも同じ公開URL、同じProject URL・公開用キー、同じアカウントを使ってログインし、「今すぐ同期」を押します。
7. 同期完了の表示を確認します。以後、記録・日まとめ・設定などを保存すると、ログイン済みで通信可能な場合に同期を試みます。必要に応じて「今すぐ同期」を押してください。

## 保存と競合について

- ログイン前やオフライン時も、記録はその端末のブラウザ内に保存されます。自動同期が成功しなかった場合は、設定画面の状態表示を確認し、オンラインで「今すぐ同期」を押してください。
- 同じ記録IDの編集は更新日時を使って統合します。削除も削除時刻を同期します。
- 重要な記録は、初回同期前と定期的にJSONバックアップを書き出してください。
- クラウド同期は1ユーザーにつき1つのデータセットを使います。複数の人で同じアカウントを共有しないでください。
- このアプリは医療記録用の保管庫や暗号化ノートではありません。特に機微な情報を保存する場合は、端末のロックとアカウント保護を有効にしてください。

## PCで使う

PCでは横幅を広く使い、左側に縦型メニューが表示されます。公開した同じURLをスマホとPCで開き、同じSupabaseアカウントでログインしてください。ローカルの `index.html` を直接開いた場合、端末間同期やPWA機能は保証されません。

## 主な機能

- 6種類の記録（日記、思い出、思考・感情、人生イベント、目標・成長、自分について）
- 日付、タイトル、本文、振り返り、気分、エネルギー、タグ、人・場所、セルフケア、明日へのメモ
- 1日に複数の個別記録と、その日のまとめ日記
- 記録の編集・削除、カレンダー、人生イベント年表、気分・タグ・記録傾向の表示
- 日替わりの問いかけ、月ごとの振り返り
- テーマ切替、文字サイズ、記録項目の表示切替
- JSONバックアップ・復元
- HTTPS配信時のPWAと基本的なオフライン表示
