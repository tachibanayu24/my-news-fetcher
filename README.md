# My News Fetcher

GitHub Actions + Claude Code で毎朝自動でニュースを収集・要約して Issue に投稿するシステム。

## ワークフロー

```mermaid
flowchart TD
    subgraph 毎朝8時 / 手動実行
        A[fetch-news.yml] --> B[Issue #1 からトピック取得]
        B --> C[likes.md / dislikes.md で好み把握]
        C --> D[Web検索で最新ニュース取得]
        D --> E[ニュースごとにIssue作成]
    end

    subgraph ユーザー操作
        E --> F[ニュースIssueを読む]
        F --> G{フィードバック}
        G -->|良い| H[👍 をつける]
        G -->|悪い| I[👎 をつける]
        H --> J[Issue を close]
        I --> J
    end

    subgraph close時
        J --> K[on-close.yml]
        K --> L{リアクション確認}
        L -->|👍| M[likes.md に追記]
        L -->|👎| N[dislikes.md に追記]
        M --> O[次回のニュース選定に反映]
        N --> O
    end
```

## 使い方

### トピックの編集

[Issue #1](https://github.com/tachibanayu24/my-neww-fetcher/issues/1) にコメント:

```
@claude ファッションも追加して
```

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

### 手動実行

Actions → Fetch Daily News → Run workflow
