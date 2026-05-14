---
name: scan-receipt
description: "レシート・領収書の画像をClaudeが読み取り、freee MCPで自動仕訳登録。電子帳簿保存法対応の画像保管も自動実施。「レシート」「スキャン」「領収書」等で起動。"
allowed-tools: Read, mcp__freee-mcp__freee_api_post, mcp__freee-mcp__freee_api_get, mcp__freee-mcp__freee_get_current_company, mcp__freee-mcp__freee_file_upload, mcp__claude_ai_Google_Calendar__list_events
---

# Scan Receipt Skill（レシートOCR自動仕訳）

レシート・領収書画像をClaudeのビジョン機能で読み取り、freee MCPで自動仕訳登録するスキル。
電子帳簿保存法対応の画像保管も自動実施。

## 前提条件

- レシート・領収書の画像ファイル（JPEG / PNG / PDF）またはカメラ撮影画像
- freee MCP 接続済み
- Google Calendar MCP（交際費・会議費の同席者補完に使用。任意）

## 処理フロー

```
Phase 1: 画像の取得・確認
    ↓
Phase 2: Claudeビジョンで情報抽出
    ↓
Phase 3: 勘定科目・品目の自動判定
    ↓
Phase 3.5: カレンダー突合（交際費・会議費のみ）
    ↓
Phase 4: 重複チェック
    ↓
Phase 5: ファイルボックスアップロード → freee登録
    ↓
Phase 6: 電子帳簿保存法対応（画像保管・リネーム）
```

## Phase 1: 画像の取得

### 対応ファイル形式

- JPEG / JPG
- PNG
- PDF

### 優先順位

1. ユーザーが明示的にパスを指定した場合 → 指定パスを使用
2. 添付ファイルがある場合 → そのまま処理
3. 複数ファイル指定可能（一括処理対応）

### 取得確認

```
対象ファイルを確認しました:
  - receipt_001.jpg
  - receipt_002.jpg

これらのレシートを処理しますか？ [y/n]
```

## Phase 2: Claudeビジョンで情報抽出

Claudeのビジョン機能を使用して各画像からテキスト情報を抽出する。

### 抽出項目

- `store_name`: 店名
- `date`: 日付（YYYY-MM-DD形式。年が省略されている場合は今年を補完）
- `total_amount`: 合計金額（整数。税込み）
- `tax_amount`: 消費税額（記載がなければnull）
- `items`: 品目リスト
- `payment_method`: 支払方法（`cash` / `credit_card` / `ic_card` / `unknown`）
- `confidence`: 読み取り信頼度（`high` / `medium` / `low`）

### OCR結果の表示

```
[読み取り結果]
  ファイル: receipt_001.jpg
  店名: スターバックスコーヒー
  日付: 2026-03-10
  金額: ¥680
  支払: クレジットカード
  信頼度: high ✅
```

信頼度が `low` の場合は内容をユーザーに確認してから次のフェーズに進む。

## Phase 3: 勘定科目・品目の自動判定

`freee-memory.md` が存在する場合は Read して学習済み取引先ルールも照合する。

### 正規化処理（マッチング前に実施）

1. 全角英数字→半角
2. 半角カタカナ→全角カタカナ
3. 英字を小文字化
4. 前後の空白トリム

### 判定ロジック

| 摘要パターン | 勘定科目 | tax_code |
|---|---|---|
| AWS / Vercel / ChatGPT 等 海外SaaS | 通信費 | 2（対象外） |
| 国内SaaS・ソフトウェア | 通信費 | 136（課税10%） |
| 飲食店（1万円以下） | 会議費 | 136 |
| 飲食店（1万円超） | 交際費 | 136 |
| 書籍・Kindle（国内） | 新聞図書費 | 163（軽減8%） |
| 書籍（海外ストア） | 新聞図書費 | 2 |
| セミナー・勉強会 | 諸会費 | 136 |
| 電車・バス・タクシー | 旅費交通費 | 136 |
| 電気・ガス・水道 | 水道光熱費 | 136 |
| 振込手数料 | 支払手数料 | 136 |

**⚠️ tax_code は必ず明示指定。`account_items` APIのデフォルト（旧5%=34）は絶対使わない。**

### フォールバック（Claude判断）

キーワードで判定できなかった場合:
```
「[店名]」はどんなお店ですか？また、何に使いましたか？
```

## Phase 3.5: カレンダー突合（交際費・会議費のみ）

交際費または会議費と判定されたレシートに対してのみ実行。

- `mcp__claude_ai_Google_Calendar__list_events` でレシート日付の予定を検索
- 候補予定を提示してユーザーが選択
- 選択した予定の件名・参加者を摘要に自動差し込み

```
🍽️ 交際費/会議費の同席者候補
  レシート: 2026-03-10 スターバックス ¥680

  該当予定:
  [1] 11:30-13:30 〇〇社打合せ / 参加: 山田さん
  [2] 予定なし（手入力）

どれに紐付けますか？ [番号]
```

カレンダーMCPが未接続の場合はこのフェーズをスキップ。

## Phase 4: 重複チェック

`freee_get_current_company` で `company_id` を動的取得してから照合。

```
GET /api/1/deals?company_id={company_id}&type=expense&start_date=YYYY-MM-DD&end_date=YYYY-MM-DD
```

**重複判定条件**（以下すべて一致で重複とみなす）:
- 日付が同一
- 金額が同一
- 摘要の先頭20文字が一致

重複が検出された場合はスキップし、結果に表示する。

## Phase 5: ファイルボックスアップロード → freee登録

### Phase 5.0: ファイルボックスへの証票アップロード

deals POST の前に必ず実施。戻り値の `receipt_id` を deals に紐付ける。

```
mcp__freee-mcp__freee_file_upload
  file_path: <ローカル画像の絶対パス>
  document_type: "receipt"
  qualified_invoice: "qualified" | "not_qualified" | "unselected"
  receipt_metadatum_issue_date: "YYYY-MM-DD"
  receipt_metadatum_amount: <金額整数>
  receipt_metadatum_partner_name: "<店名>"
```

**qualified_invoice の判定**:
- インボイス登録番号（T+13桁）を検出 → `qualified`
- 登録番号の記載なし → `not_qualified`
- 判断不能 → `unselected`

### 確認表示

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
レシートスキャン 処理結果

■ 登録候補: X件
  1. 2026/03/10 ¥680  スターバックス → 会議費（課税10%） ✅
  2. 2026/05/15 ¥3,200 アマゾン → 新聞図書費（軽減8%） ✅

■ 要確認: Y件
■ 重複スキップ: Z件

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

登録を実行しますか？ [y/n]
```

**必ずユーザーの確認を得てから登録を実行すること。**

### freee API 登録

```
POST /api/1/deals
{
  "company_id": {動的取得したcompany_id},
  "type": "expense",
  "issue_date": "YYYY-MM-DD",
  "receipt_ids": [<Phase 5.0で取得したreceipt_id>],
  "details": [{
    "account_item_id": <勘定科目ID>,
    "tax_code": <tax_code>,
    "amount": <金額>,
    "description": "<摘要>"
  }],
  "payments": [{
    "from_walletable_type": "<cash|credit_card|wallet>",
    "from_walletable_id": <口座ID>,
    "amount": <金額>,
    "date": "YYYY-MM-DD"
  }]
}
```

**セキュリティルール**:
- PUT/PATCH `/deals` は `partner_id` 必須（省略すると取引先が消える）
- 10万円超の取引は自動登録せず、必ず確認を求める
- `receipt_ids` 必須（画像なし仕訳は電子帳簿保存法の要件を満たさない）

## Phase 6: 電子帳簿保存法対応

freee登録完了後、元の画像ファイルを所定の場所へ移動・リネームする。

**保存先**: `./receipts/YYYY-MM/`

**ファイル名フォーマット**: `YYYYMMDD_店名_金額.jpg`

例: `20260516_スターバックス_680.jpg`

```
[電子帳簿保存] 画像を保管しました:
  ./receipts/2026-05/20260516_スターバックス_680.jpg ✅
```

## 複数レシートの一括処理

複数ファイルが指定された場合は順番に処理する。
1件でもエラーが発生した場合は当該ファイルのみスキップし、処理を継続する。

最後にまとめて結果を表示:
```
📊 処理完了
  登録: X件（合計 ¥XXX,XXX）
  スキップ（重複）: Y件
  要確認: Z件
```
