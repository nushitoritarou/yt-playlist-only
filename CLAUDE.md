# yt-playlist-only

スマホでYouTubeを「特定の再生リスト内の動画だけ」視聴できるようにするための、静的HTML 1枚のプロジェクト。

## 背景と目的

- YouTube標準機能では「再生リスト内の動画だけ視聴可能」にする設定は存在しない。
- そこで、再生リストをiframe埋め込みするだけの静的ページを閲覧専用の入口にする。
- スマホ本体側では通常のYouTubeアプリ/サイトをScreen Time等でブロックする。ブロック解除に必要なパスコードは第三者に預ける運用が前提。
- 参照元メモ(Notion): 「2026-10-04 スマホでYouTubeを再生リスト内の動画だけに制限する方法」の案B-0。

## 現在のスコープ(最小構成・案B-0)

- `index.html` 1枚のみ。ビルド、依存パッケージ、APIキー、cron、GitHub Actions、`videos.json` はすべて不要。
- `https://www.youtube-nocookie.com/embed/videoseries?list=<再生リストID>` をiframeで表示する。
- 動画IDの解決はYouTube側のプレイヤーが行う。再生リストに動画を追加すれば即時に反映される。
- 再生リストは「公開」または「限定公開」(非公開は埋め込み不可)。
- 再生リストIDは初回にページ内フォームで入力し、localStorageに保存する(コードに秘密情報は持たない)。

### 現在のコード

実体は `index.html`。最初の版(Notion案B-0)からの差分は次の通り。

- `<meta name="referrer" content="strict-origin-when-cross-origin">` と iframe の `referrerPolicy` を明示した(エラー153の予防)。
- 入力欄を `<form>` にして、Enterキーで保存できるようにした。
- 再生リストURLや共有リンク(`...?list=PL...&si=...`)を貼った場合は、正規表現で `list` パラメータを取り出して保存する。スキーム省略や前後に文字列が付いた貼り付けにも対応する。
- iframe の `allow` に `encrypted-media; picture-in-picture` を追加した。
- 埋め込みURLに `playsinline=1` を追加した(iOSで再生開始時にネイティブ全画面へ強制遷移するのを防ぐ)。

## 既知の問題(ここまでの検証結果)

| 事象 | 原因 | 対処 |
|---|---|---|
| `ERR_BLOCKED_BY_CSP` | Claudeのプレビュー/Artifact環境のCSPが外部ドメインのiframeを禁止している | この環境では確認不可。自前ホスティングで確認する |
| YouTubeのエラー153 | `file://` で開くとHTTPリファラが付かず、埋め込みが拒否される | `http://localhost` か https で配信して開く |
| 黒/白画面のまま | 旧版は `prompt()` を使っていた。サンドボックス環境では `prompt()` がブロックされIDが取れなかった | ページ内フォームに変更済み |
| iPhoneで再生開始と同時に全画面になり、再生リスト内の他の動画を選べない | `playsinline` の既定値は0で、iOSではネイティブ全画面再生になる。ネイティブ全画面ではYouTubeプレイヤーのUI(右上の再生リストボタン等)が表示されない | `playsinline=1` を追加した。実機での再確認待ち |
| (確認済み) フォーム→保存→iframe表示→リロード後の自動表示 | — | localhost配信とヘッドレスChromiumで動作を確認した。URLを貼った場合のID抽出も確認済み。実際の動画再生は未確認(実機確認待ち) |

## 最初にやること

1. ローカル配信で確認: `python3 -m http.server 8000` → `http://localhost:8000`
2. 153が続く場合は `<head>` に `<meta name="referrer" content="strict-origin-when-cross-origin">` を追加し、ホスティング側がno-referrerヘッダーを付けていないか確認する。
3. GitHub PagesかCloudflare Pagesにデプロイし、スマホで開く(ホーム画面に追加して使う)。
4. 次の確認項目を検証し、結果をREADMEまたはこのファイルに追記する。
   - 埋め込み禁止の動画の挙動(再生不可/スキップされる)
   - プレイヤーのロゴやタイトルからyoutube.comへ遷移できる点が許容できるか
   - スマホ本体ブロックと組み合わせて運用が回るか

## 運用上の注意(設計制約)

- このページ自体は制限として機能しない。入力UIがあるため、閲覧者は別の再生リストIDに書き換えられる。制限の実体はスマホ本体のブロック+パスコード管理。
- Safariはホーム画面に追加していないサイトのlocalStorageを、7日間アクセスがないと消す。ホーム画面に追加して使う。
- 再生リストへの動画追加自体がYouTube本体の利用になる。追加はPC側のみとし、スマホは閲覧専用にする運用が望ましい。
- `rel=0` は現仕様では「関連動画を出さない」ではなく「同じチャンネルの動画のみ表示」。使う場合はこの点に注意。

## 拡張案(最小版で不足した場合のみ)

- 案B: GitHub Actions(cron)でYouTube Data API v3の `playlistItems.list` を実行し、`videos.json`(動画IDの配列)を生成してコミット。静的HTMLが `videos.json` を読んで個別動画を埋め込み、ホワイトリスト化する。
  - APIキーはRepository Secretsに保存(`${{ secrets.YT_API_KEY }}`)。ログにキーを出さない。
  - Google Cloud ConsoleでAPI制限を「YouTube Data API v3」のみに絞る(ActionsのIPは不定なのでIP制限は不向き)。
  - クォータは `playlistItems.list` が1回1ユニット、1日10,000ユニットなので問題なし。
  - 再生リストの中身を公開したくない場合はprivateリポジトリ。ただしprivateリポジトリのGitHub Pages公開にはプラン制限があるため、Cloudflare Pages等も候補。
- 代替(別アプローチ): YouTube Kidsの「承認済みコンテンツのみ」。専用アプリで、動画/チャンネル単位のホワイトリスト。
- 不採用: オフライン再生(Premiumのダウンロードは約29日ごとにオンライン認証が必要)、ChromeのURLAllowlist(`list=` を許可しても `v=` を差し替えれば任意動画を再生できる)。

## Claude Codeへの作業指示

- まず `index.html` をこのプロジェクトのルートに作成し、上記コードで動作確認する。
- 余計な依存やビルド工程を足さない。最小構成を維持する。
- 変更した場合は、既知の問題の表と確認項目の結果をこのファイルに反映する。
- デプロイ先は未確定。GitHub PagesかCloudflare Pagesのどちらかを提案し、手順を `README.md` に簡潔にまとめる。
