---
name: freee-ws-start
description: freee × AI 会計の開始案内。freee接続を確認し、今日の作業手順を案内する。「freee-ws」「準備できた」「スタート」「始めます」等で起動。
allowed-tools: mcp__freee-mcp__freee_auth_status, mcp__freee-mcp__freee_get_current_company
---

# freee-ws-start

## Role / Inputs / Success Criteria

| 項目 | 内容 |
|------|------|
| **役割** | freee MCP の接続確認 + 今日の作業手順を案内する |
| **入力** | なし（起動するだけでOK） |
| **成功条件** | 接続中の事業所名が表示され、今日の3ステップが案内される |

## When to Use

- 「スタート」「始めます」「準備できた」と言われたとき
- freeeとの接続状態を確認したいとき
- セッション開始時に手順を確認したいとき

## ワークフロー

### Step 1: freee接続確認

`freee_auth_status` で認証状態を確認し、`freee_get_current_company` で事業所名を取得する。

### Step 2: 案内メッセージを表示

**接続OK の場合:**

```
🎉 freee AIアシスタントへようこそ！

接続中の事業所: [事業所名]

今日やること：
① CSVを添付して「仕訳して」と送る
② AIが止まったら取引先を教えてあげる
③ 全部終わったら「Memoryに覚えさせる」

まずCSVファイルを準備してください。
準備ができたら「仕訳して」と一言送るだけでOKです。
```

**接続できていない場合:**

```
freeeとの接続が確認できませんでした。

接続するには:
1. setup-guideの「MCP接続」セクションを開く
2. freeeアカウントでOAuth認証を完了する
3. 「スタート」と送ってもう一度試す
```
