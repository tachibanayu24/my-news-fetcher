# My News Fetcher

GitHub Actions + Claude Code で毎朝自動でニュースを収集・要約して Issue に投稿するシステム。

## ワークフロー

```mermaid
flowchart LR
    subgraph fetch[毎朝8時]
        T --> A[トピック取得] --> B[好み把握] --> C[Web検索] --> D[Issue作成]
    end

    subgraph user[ユーザー]
        D --> E[読む] --> F[👍 or 👎] --> G[close]
    end

    subgraph feedback[フィードバック]
        G --> H[likes/dislikes.md に記録]
    end

    H -.->|次回反映| B
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
