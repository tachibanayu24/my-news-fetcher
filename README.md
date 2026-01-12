# My News Fetcher

GitHub Actions + Claude Code で毎朝自動でニュースを収集・要約して Issue に投稿するシステム。

## 特徴

- 追加料金なし（Claude Code サブスクリプション内）
- 依存関係ゼロ（Node.js 不要）
- Issue 内で `@claude` による深堀り可能
- トピック管理も Issue で完結（`@claude` で編集可能）
- 👍👎 フィードバックで好みを学習

## セットアップ

### 1. シークレットの設定

GitHub リポジトリの Settings → Secrets and variables → Actions で以下を設定:

| シークレット名            | 説明                                            |
| ------------------------- | ----------------------------------------------- |
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code サブスクリプションの OAuth トークン |

### 2. TARGET_TOPICS Issue の作成

```bash
gh issue create --title "TARGET_TOPICS" --body "$(cat <<'EOF'
- LLM・生成AI
  - 特に新モデルのリリース、ベンチマーク結果
- スタートアップ資金調達
  - シリーズA以上、調達額1億円以上
- Web Frontend
- 日本企業IPO
EOF
)"
```

## 使い方

### トピックの追加

Issue #1 にコメント:

```
@claude ファッションも追加して
```

### トピックの削除

```
@claude Web Frontend を削除して
```

### ニュースの深堀り

ニュース Issue にコメント:

```
@claude この資金調達について、投資家の詳細を教えて
```

### フィードバック

ニュース Issue に 👍 or 👎 をつけてから close

## 手動実行

GitHub Actions の「Fetch Daily News」ワークフローから「Run workflow」で手動実行可能。
