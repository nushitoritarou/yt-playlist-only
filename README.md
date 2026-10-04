# yt-playlist-only

YouTubeの特定の再生リストだけを埋め込み表示する、静的HTML 1枚のページです。設計の背景や制約は [CLAUDE.md](CLAUDE.md) を参照してください。

## ローカルで確認する

```sh
python3 -m http.server 8000
# http://localhost:8000 を開く
```

`file://` で直接開くとリファラが付かず、YouTubeがエラー153を返します。必ずHTTPで配信してください。

## 使い方

1. 初回だけ、再生リストID（`PL...`）か再生リストのURL（`...?list=PL...`）を入力して保存します。
2. 値はブラウザのlocalStorageに保存され、次回からは再生リストがすぐに表示されます。
3. iPhoneでは、Safariの共有メニューから「ホーム画面に追加」して使ってください。ホーム画面に追加していないと、7日間アクセスがない時点で保存したIDが消えます。

再生リストは「公開」または「限定公開」にしておきます（「非公開」は埋め込めません）。

## デプロイ（GitHub Pagesを推奨）

このリポジトリはすでにGitHubにあり、ビルド工程もないため、GitHub Pagesがいちばん手数が少ない方法です。

1. リポジトリの **Settings → Pages** を開きます。
2. **Source** で「Deploy from a branch」を選びます。
3. **Branch** で `main`（または公開に使うブランチ）と `/ (root)` を選び、Save します。
4. 数分後に表示される `https://<user>.github.io/yt-playlist-only/` をスマホで開き、ホーム画面に追加します。

GitHub Pagesはリファラを消すヘッダーを付けないので、エラー153の心配はありません。

リポジトリをprivateにしたまま公開したい場合、GitHub Freeプランでは使えません。その場合はCloudflare Pagesを使います（Workers & Pages → Create → Pages → Gitリポジトリを接続し、ビルドコマンドは空、出力ディレクトリは `/`）。なお、再生リストIDはコードに含まれないので、リポジトリをpublicにしても再生リストの中身は公開されません。
