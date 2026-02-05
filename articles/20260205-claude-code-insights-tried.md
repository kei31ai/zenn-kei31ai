---
title: "Claude Codeの /insights を実際に使ってみた—4ヶ月分の使用パターンを丸裸にされた"
emoji: "📊"
type: "tech"
topics: ["claudecode", "claude", "ai", "開発ツール"]
published: true
---

先ほど「[Claude Codeの /insights コマンドとは？](https://zenn.dev/kei31ai/articles/20260205-claude-code-insights-command)」という記事を書いたのですが、実際に自分の環境で動かしてみました。結果、4ヶ月分の使用データを分析されて、かなり的確なフィードバックをもらいました。

## 実行してみた

Claude Codeで `/insights` と入力するだけ。数秒でHTMLレポートが生成されました。

![レポート全体](/images/20260205-claude-code-insights-tried/report-overview.jpg)

## ぼくの使用統計

分析されたデータはこんな感じでした。

- **セッション数**: 7,198
- **メッセージ数**: 77,582
- **期間**: 2025年10月12日〜2026年2月5日（約4ヶ月）
- **コミット数**: 356

ツール使用回数も詳細に出てきます。

- **Bash**: 65,000回以上
- **Read**: 51,000回以上
- **Edit**: 48,000回以上
- **Write**: 17,000回以上
- **Serena MCP（replace_content）**: 7,400回以上

## 「うまくいっていること」の分析

レポートは「Heavy Command-Line Integration」（コマンドライン統合が優れている）と評価してくれました。

> You've executed over 65,000 Bash commands through Claude, making it a true extension of your terminal workflow.

65,000回以上のBashコマンドを実行していて、Claude Codeをターミナルの延長として使いこなしている、と。

また「Read-Edit-Writeのバランスが良い」とも指摘されました。51K読み込み、48K編集、17K書き込みという比率は「コードを十分に理解してから変更する」アプローチを示しているそうです。

## 「課題」の分析—これが痛かった

ここからが本題。ぼくの弱点を的確に指摘されました。

**1. デザインリクエストの仕様不足**

> You're starting design requests without providing enough context about your specific vision

YouTubeサムネイルを作ろうとしたとき、「eye-catchingなスタイルにしてほしい」と後から指定し直す必要があった、と具体例まで出されました。確かにそういうことがあった。

**2. セッションの途中終了**

> You're ending sessions before tasks finish, particularly when Claude is setting up tools or skills needed for your request

「サムネイルスキルをインストール中にセッションが終わった」という例を挙げられました。7,198セッションで356コミットということは、多くのセッションが具体的なアウトプットに到達していない、と。

**3. ビジュアル作業とツールのミスマッチ**

> Your top tools are Bash (65K uses) and code editing tools, but your top goal is 'design_graphics'—there's a fundamental mismatch

コード系ツールでグラフィック作業をしようとしている根本的なミスマッチがある、と指摘されました。

## 提案された改善策

具体的な改善案も出てきました。

**カスタムスキルの作成**

```bash
mkdir -p .claude/skills/thumbnail && cat > .claude/skills/thumbnail/SKILL.md << 'EOF'
# YouTube Thumbnail Creator

Create eye-catching YouTube thumbnails with:
- Dimensions: 1280x720 pixels
- Bold, readable text with contrasting colors
- High-contrast imagery that pops at small sizes
- Entertainment/clickbait style unless specified otherwise

Ask user for: video topic, main text, preferred colors/mood
EOF
```

デザインの好みを事前にスキルとして定義しておけば、毎回説明し直す必要がなくなる、という提案。

**MCPサーバーで画像生成APIに接続**

```bash
claude mcp add image-gen -- npx @anthropic/mcp-image-generator
```

コード系ツールでビジュアル作業をするミスマッチを解消するには、画像生成APIに直接接続すべき、という提案。

## 「将来の可能性」—ここが面白い

レポートの最後には「On the Horizon」というセクションがあり、ぼくの使い方から見た将来の可能性を提案してくれました。

**自律的なテスト駆動開発ループ**

> Have Claude autonomously iterate on implementations until all tests pass, handling failures, debugging, and refactoring without human intervention.

テストが通るまでClaudeが自律的にコードを修正し続ける、という使い方。65K回以上のBash実行経験があるなら、そこまで任せられるはず、と。

**並列エージェントアーキテクチャ**

> Orchestrate multiple specialized agents working simultaneously—one on frontend, one on backend, one on tests, one on documentation—all coordinating through shared files.

フロントエンド、バックエンド、テスト、ドキュメントを担当する複数のエージェントを並列で動かす、という構想。ぼくのTodoWrite使用（4,700回以上）が既にその土台になっている、と分析されました。

## 感想

正直、かなり的確だった。「セッションを途中で終わりすぎ」「デザインリクエストの仕様が足りない」という指摘は、言われてみれば確かにそう。

特に印象的だったのは、具体的なセッションの例を挙げて指摘してくること。「YouTubeサムネイルで後からスタイルを指定し直した」とか、実際にあった出来事をちゃんと覚えている。

統計の数字が正確かどうかは検証していませんが、ナラティブな分析（どこで詰まっているか、何がうまくいっているか）は十分に参考になりました。

Claude Codeをある程度使い込んでいる人は、一度試してみる価値があると思います。自分の使い方の癖が見えてきます。

## 実行方法

Claude Codeで以下を入力するだけです。

```
/insights
```

レポートは `~/.claude/usage-data/report.html` に保存されます。

**関連記事:**
- [Claude Codeの /insights コマンドとは？](https://zenn.dev/kei31ai/articles/20260205-claude-code-insights-command)

**公式リソース:**
- [Claude Code GitHub リポジトリ](https://github.com/anthropics/claude-code)
- [公式CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
