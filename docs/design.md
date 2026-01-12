# My News Fetcher - 設計ドキュメント

## 概要

GitHub Actions + Claude Code を使い、毎朝自動でニュースを収集・要約して Issue に投稿するシステム。

**特徴**:

- 追加料金なし（Claude Code サブスクリプション内）
- 依存関係ゼロ（Node.js 不要）
- Issue 内で `@claude` による深堀り可能
- トピック管理も Issue で完結（`@claude` で編集可能）
- **👍👎 フィードバックで好みを学習**

**運用**:

- ニュースを読む → 👍 or 👎 → close
- フィードバックは自動で記録され、次回以降のニュース選定に反映

---

## ファイル構成

```
my-news-fetcher/
├── .github/
│   └── workflows/
│       ├── fetch-news.yml      # 毎朝ニュース取得
│       ├── claude-respond.yml  # @claude 応答
│       └── on-close.yml        # close時フィードバック記録
├── feedback/
│   ├── likes.md                # 👍 したニュース
│   └── dislikes.md             # 👎 したニュース
├── docs/
│   └── design.md
└── README.md
```

---

## トピック管理（Issue #1: TARGET_TOPICS）

### Issue 本文のフォーマット

```markdown
- LLM・生成AI
  - 特に新モデルのリリース、ベンチマーク結果
- スタートアップ資金調達
  - シリーズA以上、調達額1億円以上
- Web Frontend
- 日本企業IPO
```

### トピック編集の例

```
@claude ファッションも追加して。ハイブランドの動向を優先で。
```

---

## フィードバック管理

### likes.md / dislikes.md のフォーマット

```markdown
- タイトル | URL
- OpenAIがGPT-5を発表 | https://example.com/1
- LayerX、100億円調達 | https://example.com/2
```

### 運用フロー

```
1. ニュース Issue を読む
2. 👍 or 👎 リアクションをつける
3. Issue を close
4. 自動で likes.md or dislikes.md に追記・コミット
5. 次回 fetch-news 時にフィードバックを参照
```

---

## 出力フォーマット

### Issue タイトル

```
📰 ニュース内容の要約（30文字以内）
```

例:

- `📰 OpenAIがGPT-5を発表、処理速度2倍に`
- `📰 LayerX、シリーズCで100億円調達`
- `📰 フリー、東証グロース市場に上場`

### Issue 本文

```markdown
**サマリー**:
OpenAIは新しいGPT-5モデルを発表。従来比2倍の処理速度と
マルチモーダル機能の強化が特徴。企業向けAPI提供も開始予定。

**URL**: https://example.com/news/1

**ポイント**:

- 処理速度2倍
- マルチモーダル強化
- 企業向けAPI提供
```

### フォーマット仕様

| 項目     | 制約                            |
| -------- | ------------------------------- |
| タイトル | 30 文字以内（ニュース内容要約） |
| サマリー | 150 文字以内                    |
| URL      | 元記事へのリンク                |
| ポイント | 箇条書き 3 つ程度               |
| 件数     | トピックあたり最大 5 件         |

---

## GitHub Actions

### 1. fetch-news.yml（毎朝ニュース取得）

```yaml
name: Fetch Daily News

on:
  schedule:
    - cron: "0 23 * * *" # JST 08:00
  workflow_dispatch: # 手動実行

jobs:
  fetch-news:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      issues: write
      id-token: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v5
        with:
          fetch-depth: 1

      - name: Run Claude Code
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          claude_args: |
            --model claude-opus-4-5-20251101
            --allowedTools "WebSearch,Read,Bash(gh issue view:*),Bash(gh issue create:*)"
          prompt: |
            1. Issue #1 (TARGET_TOPICS) を取得:
               gh issue view 1 --json body

            2. feedback/likes.md と feedback/dislikes.md を読んで好みを把握

            3. 各トピックについて今日の最新ニュースをWeb検索で取得
               - likes.md の傾向に合うニュースを優先
               - dislikes.md の傾向に合うニュースは除外

            4. 以下のフォーマットでIssueを作成:
               【タイトル】📰 ニュース内容の要約（30文字以内）
               【本文】
               - サマリー: 150文字以内
               - URL: 元記事へのリンク
               - ポイント: 箇条書き3つ

            5. Issue作成:
               gh issue create --title "📰 要約" --body "..."
```

### 2. on-close.yml（close時フィードバック記録）

```yaml
name: Record Feedback

on:
  issues:
    types: [closed]

jobs:
  record-feedback:
    if: startsWith(github.event.issue.title, '📰')
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: read
      id-token: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v5

      - name: Run Claude Code
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          claude_args: |
            --model claude-opus-4-5-20251101
            --allowedTools "Read,Write,Bash(git add:*),Bash(git commit:*),Bash(git push:*),Bash(gh api:*)"
          prompt: |
            closeされたIssueのリアクションを確認してフィードバックを記録:

            1. Issueのリアクションを取得:
               gh api repos/${{ github.repository }}/issues/${{ github.event.issue.number }}/reactions

            2. リアクションに応じて処理:
               - 👍 (+1) があれば feedback/likes.md に追記
               - 👎 (-1) があれば feedback/dislikes.md に追記
               - どちらもなければ何もしない

            3. 追記フォーマット:
               - Issueタイトル | URL（本文から抽出）

            4. 変更があればコミット＆プッシュ:
               git add feedback/
               git commit -m "Record feedback for #${{ github.event.issue.number }}"
               git push
```

### 3. claude-respond.yml（@claude 応答）

```yaml
name: Claude Respond

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review' && contains(github.event.review.body, '@claude')) ||
      (github.event_name == 'issues' && (contains(github.event.issue.body, '@claude') || contains(github.event.issue.title, '@claude')))
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
      issues: write
      id-token: write
      actions: read
    steps:
      - name: Checkout repository
        uses: actions/checkout@v5
        with:
          fetch-depth: 1

      - name: Run Claude Code
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          claude_args: |
            --model claude-opus-4-5-20251101
            --allowedTools "WebSearch,Read,Grep,Glob,Bash(gh issue edit:*),Bash(gh issue view:*)"
```

---

## 認証設定

### 必要なシークレット

| シークレット名            | 説明                                            |
| ------------------------- | ----------------------------------------------- |
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code サブスクリプションの OAuth トークン |

### 設定手順

1. GitHub リポジトリの Settings → Secrets and variables → Actions
2. `CLAUDE_CODE_OAUTH_TOKEN` を追加
3. Claude Code のダッシュボードからトークンを取得して設定

---

## 初期セットアップ

```bash
# Issue #1 を作成
gh issue create --title "TARGET_TOPICS" --body "$(cat <<'EOF'
- LLM・生成AI
- スタートアップ資金調達
- Web Frontend
- 日本企業IPO
EOF
)"

# フィードバックファイルを作成
mkdir -p feedback
echo "# Likes" > feedback/likes.md
echo "# Dislikes" > feedback/dislikes.md
git add feedback/
git commit -m "Initialize feedback files"
git push
```

---

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

---

## コスト見積もり

| 項目         | 値              |
| ------------ | --------------- |
| 実行頻度     | 1 日 1 回       |
| トピック数   | 4 件            |
| モデル       | claude-opus-4-5 |
| 推定トークン | 約 20,000/日    |
| 月間トークン | 約 600,000/月   |
| @claude 応答 | 使用量による    |

**結論**: Claude Code サブスクリプションの範囲内で運用可能（Opus は Max プラン推奨）

---

## 参考

- [Claude Code Action](https://github.com/anthropics/claude-code-action)
- [GitHub Actions scheduled events](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule)
