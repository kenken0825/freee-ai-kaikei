---
name: freee-check
description: "freeeの未処理明細をチェックし、状況をサマリー表示する。取引品質アラート・wallet_txn乖離検出付き。「経理チェック」「freee確認」「未処理」等のトリガーで起動。"
allowed-tools: mcp__freee-mcp__freee_auth_status, mcp__freee-mcp__freee_api_get, mcp__freee-mcp__freee_get_current_company, mcp__freee-mcp__freee_api_put
---

# freee Check Skill

freee MCPを使って未処理の取引明細をチェックし、サマリーを表示するスキル。

## ワークフロー

### Step 1: 認証状態の確認

```
freee_auth_status
```

認証が切れている場合は再認証を促す。

### Step 2: 事業所取得

```
freee_get_current_company → company_id と事業所名を動的取得
```

### Step 3: 未処理明細の取得

```
GET /api/1/wallet_txns?company_id={company_id}&status=1
```

- status=1: 未処理の明細
- 出金（`entry_side=credit`）と入金を分けて集計

### Step 4: 直近の取引一覧の取得

```
GET /api/1/deals?company_id={company_id}&limit=20&type=expense
```

- 直近20件の経費取引を取得
- 登録日・金額・勘定科目・摘要をサマリー表示

### Step 5: サマリー表示

```
📊 freee経理状況サマリー
接続中の事業所: [事業所名]

未処理明細: X件
  - 出金: Y件（合計 ¥ZZZ,ZZZ）
  - 入金: W件（合計 ¥VVV,VVV）

直近の登録済み取引（最新5件）:
  - 2026/05/10 通信費 ¥3,200 AMAZON WEB SERVICES
  - 2026/05/09 会議費 ¥880  スターバックス
  ...

⚠️ 注意事項:
  - 未処理30日以上の明細: N件
```

### Step 5.5: 登録済みdealsの品質チェック

直近deals の以下の問題を検出して表示する。

| チェック | 判定 | アクション |
|---|---|---|
| `partner_id` 欠損 | null または未設定 | 要確認リストへ |
| `tax_code` 欠損/不整合 | null または科目と矛盾 | HITLで確認 |
| 重複 deals | 同一 `issue_date` + `amount` + 摘要先頭20文字が2件以上 | 片方削除候補として表示 |
| 滞留取引 | `issue_date` が60日以上前かつ未振替 | 注意喚起 |

```
⚠️ 取引品質アラート

partner_id 欠損: 3件
  - 2026/04/10 ¥3,200 アマゾン

重複疑い: 1組
  - 同日同額の取引が2件存在

60日以上滞留: 2件
  - 2026/02/10 ¥15,000（振替未完了）
```

問題なければ「✅ 取引品質に問題はありません」と表示。

### Step 6: wallet_txn ⇄ deals の乖離検出

deals 登録済みだが wallet_txn が未消込（status=1 のまま）のケースを検出する。

```
GET /api/1/wallet_txns?company_id={company_id}&status=1&start_date={当月初}&end_date={今日}
GET /api/1/deals?company_id={company_id}&type=expense&start_date={当月初}&end_date={今日}
```

**検出ロジック**: 各未処理 wallet_txn について、deals の中に「日付±1日 かつ 金額一致」のものがあれば乖離候補として表示。

```
⚠️ wallet_txn未消込だがdeals登録済みの疑い: N件
  1. 2026/05/10 ¥3,200 アマゾン → deals#XXXXX と一致

一括で wallet_txn を消込済みに更新しますか？ [y/n]
```

ユーザー承認後、該当 wallet_txn を `status=2` に一括更新。

## 出力

- コンソールにサマリーを表示（ファイル出力なし）
- 問題がある場合は対応アクションを提案

## セキュリティ

- 取引の詳細（摘要・金額）はサマリーレベルでのみ表示
- APIキー等の認証情報は表示しない
