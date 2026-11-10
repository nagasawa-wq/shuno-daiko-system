# 収納代行 管理システム（nsd-invoice）

決済代行各社の入金明細PDFを読み込み、サイト別の売上集計・会社間の請求書発行・freee用の仕訳CSV作成を行う社内ツールです。

- 本番URL: https://nsd-invoice.vercel.app
- ホスティング: Vercel（チーム `banananan` / プロジェクト `nsd-invoice`）
- データ保存先: Firebase Realtime Database（プロジェクト `nsd-invoice`）
- 構成: ビルド不要の静的HTMLのみ（サーバー側のコード・APIなし）

> このリポジトリは 2026-11-10 に、Vercel の本番デプロイ（`dpl_F4owWyFdg7gi9SM4kHMb4UAWMxmc`、2026-11-04 作成）からソースを取り出して作成しました。それ以前のGit履歴はありません。

---

## ファイル構成

| ファイル | 役割 |
|---|---|
| `index.html` | メイン画面。タブ①〜⑥すべてを含む。Firebase との読み書きもここ |
| `jiwake.html` | タブ⑥「仕訳変換」の中身（`index.html` から iframe で読み込み） |
| `univapay.html` | 旧「⑦ ユニヴァペイ 仕訳変換」。現在は `jiwake.html` に統合済みで、メイン画面からは非表示。URL直打ちでのみ使用可 |
| `data/` | Firebase のバックアップ置き場（**Git管理外**。`.gitignore` 参照） |

外部ライブラリはすべて CDN から読み込んでいます（PDF.js 3.11.174、Firebase JS SDK 10.12.2 compat、Google Fonts）。

---

## 画面の使い方

画面上部で**対象月**を選ぶと、その月のデータが Firebase から読み込まれます。入力内容は自動保存されます（右上に「同期済み ✓」と表示）。

| タブ | 内容 |
|---|---|
| ① データ入力 | サイト × 決済手段ごとの売上金額を入力・確認する表 |
| ② PDFアップロード | 決済代行会社の明細PDFをドラッグ＆ドロップすると、金額を読み取って①に反映。取り込み履歴も表示 |
| ③ 集計表（請求分） | 会社間で請求が発生するサイトの集計。印刷可 |
| ④ 集計表（自社分） | 自社完結サイトの集計、リミットからの銀行振込の分配。印刷可 |
| ⑤ 請求書 | 請求関係ごとの請求書を表示・印刷（発行日は翌月1日が初期値） |
| ⑥ 仕訳変換 | ロボペイ／ユニヴァペイ／SUI の明細PDFから freee インポート用 CSV を作成 |

### PDF読み取りに対応している明細
SUI（プライムバンテージ）、ロボットペイメント（ロボペイ）、ユニヴァペイ（UPC）、マックスコネクト、CREDIX、ペイメントフォー（メタップス）、LINQSEA

明細のレイアウトが変わると読み取れなくなります。その場合は `index.html` の `parseSUI` / `parseRobotPayment` / `parseUPC` / `parseMaxConnect` / `parseCREDIX` / `parsePaymentFor` / `parseLINQSEA` を修正してください。

---

## よくある修正の場所（すべて `index.html`）

| やりたいこと | 修正する定数 |
|---|---|
| 会社の住所・電話番号・振込先口座の変更 | `COMPANIES` |
| 請求関係（どの会社がどの会社に請求するか）の追加 | `RELS` |
| 請求ありサイトの追加・変更 | `SITES`（`rel` は `RELS` の `id`） |
| 自社完結サイトの追加・変更 | `SELF_SITES` |
| リミットからの銀振分配グループ | `LIMIT_GROUPS` |
| 決済手段の列の追加 | `COLS` |
| PDF上の店舗名・ID とサイトの対応 | `SUI_SITE_MAP` / `RP_STORE_MAP` / `UPC_NAME_MAP` / `PMF_STORE_MAP` |

サイトを追加したら、PDFの店舗名との対応表（上記 `*_MAP`）も合わせて更新してください。SUI の対応は画面からも登録でき、Firebase の `suiMapping` に保存されます。

---

## データ（Firebase Realtime Database）

URL: `https://nsd-invoice-default-rtdb.asia-southeast1.firebasedatabase.app`

| パス | 中身 |
|---|---|
| `data/<YYYY-MM>/<サイトID>/<列ID>` | 月別・サイト別・決済手段別の金額 |
| `pdfHistory/<YYYY-MM>/<タイムスタンプ>` | PDF取り込み履歴 |
| `suiMapping/<SUI店舗ID>` | SUI の店舗ID → サイトIDの対応 |

その他、選択中の月はブラウザの localStorage（`selectedMonth`）に保存されます。

### バックアップ
WSL などで次を実行すると、DB全体を JSON で保存できます（`data/` は Git に上げません）。

```bash
mkdir -p data
curl -s "https://nsd-invoice-default-rtdb.asia-southeast1.firebasedatabase.app/.json?format=export" \
  -o "data/firebase-rtdb-$(date +%Y%m%d-%H%M%S).json"
```

※ 上記はDBが認証なしで読める現状を前提にしています。下記のセキュリティ対応でルールを厳しくした後は、Firebase コンソール（Realtime Database →「︙」→「JSONをエクスポート」）から取得してください。

---

## デプロイ（Vercel）

Git 連携はしておらず、Vercel CLI で直接デプロイしています。

```bash
# 初回のみ: このフォルダを既存の Vercel プロジェクトに紐づける
vercel link            # チーム banananan / プロジェクト nsd-invoice を選択

# 本番に反映
vercel --prod
```

ビルド設定は不要です（Framework: Other、Build Command なし、Output はフォルダの一番上）。
今後は GitHub 連携（Vercel ダッシュボード → Settings → Git）にすると、push だけで反映でき、履歴も残るのでおすすめです。

---

## ⚠ セキュリティ上の注意（要対応）

1. **Firebase DB にログインなしでアクセスできる状態です。** アプリはログイン機能を使っておらず、2026-11-10 時点で認証なしで全データを読み出せることを確認しました。書き込みも可能な可能性が高いです。
   - 対応案: Firebase Authentication（Googleログイン等）をアプリに追加し、Realtime Database のルールをログイン済みユーザーのみに制限する。ルールだけ先に変えるとアプリが動かなくなるので、必ずセットで行うこと。
2. **`index.html` に会社の住所・電話番号・振込先口座が書かれています。** このリポジトリは必ず **Private** のままにしてください。また、本番URLを知っていれば誰でも閲覧できる点にも注意。
3. DBのバックアップファイル（`data/`）には請求データが含まれます。Git には上げず、共有する場合も取り扱いに注意してください。
