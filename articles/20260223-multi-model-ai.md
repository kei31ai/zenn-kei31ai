---
title: "Claude CodeからCodex・Gemini・GLMに外注する「セカンドオピニオン」設計"
emoji: "🤝"
type: "tech"
topics: ["claudecode", "ai", "aiagent", "llm"]
published: true
---

:::message
この記事は「Claude Codeで自律型AIエージェントを作る」シリーズ第12回です。
第11回: [スキルを87個運用したら898KBになった。SKILL.md分割で27KBに戻した実測記録](https://zenn.dev/kei31ai/articles/20260221-skill-context-optimization)
:::

Claude CodeをメインのAIエージェントとして運用しつつ、Codex・Gemini・MiniMax・GLMの4モデルに「セカンドオピニオン」を外注する設計を実装した。記事の外部採点、壁打ち、ファクトチェックを複数モデルで並列処理する仕組みの設計思想と実装コード、運用データを公開する。

> **検証環境:** macOS Sequoia 15.5 / Claude Code（Claude Opus 4.6）/ Gemini CLI v0.29.0 / Codex CLI / OpenCode CLI / 検証日: 2026-02-23

## マルチモデル活用とは

マルチモデル活用とは、1つのAIエージェントが複数のLLMを目的に応じて使い分ける設計パターンのことです。

1つのモデルだけでは限界があります。自分で書いた文章を自分でレビューしても盲点に気づけない。特定の領域（最新情報、コード検証など）が苦手なモデルもある。1つのモデルの「癖」に偏った出力になりがちです。

医師が診断に迷ったときにセカンドオピニオンを求めるように、AIエージェントも別のAIに意見を聞く仕組みがあると品質が上がります。

## アーキテクチャ全体像

ぼくのAIエージェントは以下の構成で動いています。

**メインエージェント: Claude Code（Claude Opus 4.6）**

全タスクの中心です。記事執筆、SNS投稿、プロジェクト管理、コード生成を担当します。`.claude/skills/` にスキルファイル（タスクの手順書）を配置し、パイプライン（複数のタスクを順番に実行する仕組み）として動作します。

**サブモデル: 4つの外部AI**

- **[Gemini CLI](https://github.com/google-gemini/gemini-cli)**: Google Gemini 3。Web検索・最新情報に強い。無料
- **[Codex CLI](https://developers.openai.com/codex/cli/)**: OpenAI GPT-5.3-Codex。コーディング・技術分析が得意
- **[MiniMax M2.5](https://www.minimax.io/news/minimax-m25)**: 230B MoE（Mixture of Experts: 全パラメータのうち10Bだけを選択的に使う軽量設計）モデル。低コスト。OpenRouter経由
- **[GLM-5](https://www.zhipuai.cn/en)**: Zhipu AI。744B MoEモデル（44Bアクティブ）。MITライセンス

**Claude Code自身がこれらのCLIツールを呼び出す**のがポイントです。人間が4つのターミナルを開くのではなく、スキルとして自動化しています。

![マルチモデルアーキテクチャ全体像](/images/20260223-multi-model-ai/architecture.jpg)

## 4モデルへの接続コード

各モデルへの接続は専用スキルで定義しています。

```bash
# Gemini CLI（task-query-gemini）
gemini -p "質問内容" -o json 2>&1
# → response フィールドに回答

# Codex CLI（task-query-codex）
codex exec --skip-git-repo-check --json "質問内容" 2>&1
# → item.type: "agent_message" の item.text に回答

# MiniMax M2.5（task-query-minimax）
opencode run --format json --dir /tmp -m openrouter/minimax/minimax-m2.5 "質問内容" 2>&1

# GLM-5（task-query-glm）
opencode run --format json --dir /tmp -m zai/glm-5 "質問内容" 2>&1
# → type: "text" の part.text に回答
```

MiniMaxとGLMは[OpenCode CLI](https://github.com/nicepkg/opencode)経由で接続しています。

> **補足:** opencode実行時は `--dir /tmp` が必須です。プロジェクトディレクトリで実行すると、`.claude/skills/` のスキルファイルを読み込んで再帰的にパイプラインを呼び出す問題が発生します。

## セカンドオピニオン戦略: 外部採点の設計

マルチモデルの最大の活用先が「外部採点」です。Zenn記事パイプライン（`pipeline-zenn-article`）のPhase 7では、書き終わった記事を4モデルに同時に送って採点してもらいます。

**なぜClaude Code自身は採点しないのか**

自分が書いた記事を自分で採点しても意味がありません。同じモデルが書いて同じモデルがレビューしたら、同じバイアスで「良い」と判断してしまう。それぞれのモデルが異なる学習データ、異なるアーキテクチャ、異なるバイアスを持っているからこそ、多角的なレビューになります。

なお、Claude Code内からClaude自身を呼び出すことはできません（`claude -p` はネスト不可）。これは制約ではなく、「自分で自分をレビューしない」を強制してくれる良い制約です。

**採点基準:** 8項目（明確さ15点、構造10点、具体性15点、読みやすさ10点、ファクトチェック15点、SEO 15点、LLMO 10点、図解10点）の100点満点。全モデルに同一プロンプトを送ります。

**結果の統合ルール:**

1. 4つの結果が揃ったら、各項目ごとにスコアを並べる
2. **2者以上が共通して指摘した改善点は必ず修正する**
3. 1者だけの指摘は妥当性を判断して対応する
4. 修正したら完了。再採点ループはしない

この「2者以上の共通指摘」ルールがポイントです。1つのモデルだけの指摘は「そのモデルの癖」かもしれない。でも2つ以上が同じことを言っているなら、それは本当の問題です。

![セカンドオピニオンのフロー](/images/20260223-multi-model-ai/second-opinion-flow.jpg)

## 実際の運用データ

直近のPodcast原稿（2026/2/23）での外部採点結果です。

- **Gemini: 95/100**
- **GLM: 90/100**
- **MiniMax: 83/100**
- **Codex: 78/100**
- **4者平均: 86.5/100**

モデルによって最大17点の差があります。Codexが78点と最も低い評価をつけた場合、Codexの指摘をGeminiやGLMの指摘と照合します。共通していれば確実に修正すべき箇所。Codexだけの指摘ならモデル固有の基準かもしれないので判断します。

Zenn記事でも同様です。第11回記事の採点では、4者とも「図解が足りない」と指摘しました。全員一致なので迷わず図解を追加しました。

## 汎用クエリ: pipeline-query-all

外部採点以外にも、`pipeline-query-all` で1つの質問を4モデルに同時に投げられます。ブレインストーミング、技術選定、ファクトチェックなど「1つの正解がない問い」に有効です。

4モデルが違うことを言ったら、そこに深掘りすべきポイントがある。全員が同じことを言ったら、信頼度が高い情報です。

## 実装の注意点

**コスト:**

- Gemini CLI: 無料（Googleアカウント）
- Codex CLI: ChatGPT Plus/Proに含まれる（追加課金なし）
- MiniMax M2.5: OpenRouter経由。Claude Opus 4.6の約1/20のコストとされています
- GLM-5: OpenRouter経由。$0.80/M入力トークン程度

4モデル合計でも1回のクエリにかかるコストは数円です。

**タイムアウト:** 4モデル全てが返ってこないこともあります。その場合は「返ってきた分だけで集約する」がルールです。

**各モデルの得意領域:**

- **Gemini**: 最新Web情報に基づく事実確認
- **Codex**: コードスニペットの正確性チェック
- **MiniMax**: 大量テキストのコスパ良い処理
- **GLM**: 欧米モデルとは異なる視点

## Q&A

**Q: なぜ4モデルも使うのですか? 2つで十分では?**

A: 「2者以上の共通指摘を修正する」ルールを機能させるには最低3モデルが必要です。4モデルにしているのは、1つがタイムアウトしても3つで判断できる冗長性のためです。

**Q: セカンドオピニオンは品質にどのくらい影響しますか?**

A: 定量比較データはありません。ただ、外部採点を導入してから「ファクトの誤り」「読みにくい構成」「SEO対策の漏れ」が採点段階で検出されるようになり、公開後の修正が減りました。

## まとめ

- **設計思想**: 自分で書いて自分でレビューしない。外部モデルにセカンドオピニオンを外注する
- **接続方法**: Gemini CLI / Codex CLI / OpenCode CLI（MiniMax・GLM）の3つのCLIツール
- **統合ルール**: 2者以上の共通指摘は必ず修正する。1者だけの指摘は判断する
- **コスト**: 4モデル合計で1クエリ数円。Gemini・Codexは実質無料

1つのモデルに全てを任せる時代から、複数モデルを使い分ける時代に移っています。企業レベルでは37%が5つ以上のモデルを本番利用しているとされています（[参考](https://research.aimultiple.com/llm-orchestration/)）。個人開発者でも、CLIツールを組み合わせれば同じことができます。

次回（第13回）は「自律型AIエージェントの未来」をテーマに、シリーズ全体を振り返りながらAIキャラクターが社会に溶け込む世界について考えます。

**参考リンク:**

- [Claude Code Sub-agents（公式ドキュメント）](https://code.claude.com/docs/en/sub-agents)
- [Gemini CLI（GitHub）](https://github.com/google-gemini/gemini-cli)
- [Codex CLI（OpenAI公式）](https://developers.openai.com/codex/cli/)
- [MiniMax M2.5（公式発表）](https://www.minimax.io/news/minimax-m25)
- [GLM-5（Z.ai公式）](https://www.zhipuai.cn/en)
- [LLM Orchestration in 2026（AIMultiple）](https://research.aimultiple.com/llm-orchestration/)
