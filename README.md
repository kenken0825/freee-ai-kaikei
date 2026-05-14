# freee × AI 会計プラグイン

Claude Code 用プラグイン。freee MCP と連携して経理業務を AI で自動化します。

## インストール

```bash
claude plugin marketplace add https://github.com/kenken0825/freee-ai-kaikei
claude plugin install freee-ai-kaikei
```

## 含まれるスキル

| スキル | 起動ワード | 機能 |
|--------|-----------|------|
| `freee-ws-start` | 「スタート」「始めます」 | freee接続確認・今日の作業手順案内 |
| `csv-kaikei` | 「仕訳して」「CSVを処理して」 | CSV明細を一括仕訳・freee登録 |
| `scan-receipt` | 「レシート」「スキャン」 | レシート画像OCR→自動仕訳・電子帳簿保存 |
| `freee-check` | 「チェック」「確認して」 | 登録済み取引の品質チェック |
| `freee-memory` | 「覚えておいて」「Memoryに入れて」 | 仕訳ルール・会社情報をローカル保存 |
| `keihi-checklist` | 「チェックリスト」「月次」 | 月次経費チェックリスト生成 |
| `tax-risk-check` | 「税務チェック」「リスク確認」 | 7カテゴリの税務リスクスキャン・HTMLレポート出力 |

## 前提条件

- [freee MCP](https://mcp.freee.co.jp/) の接続設定済み
- freee アカウント（30日間無料トライアル可）
