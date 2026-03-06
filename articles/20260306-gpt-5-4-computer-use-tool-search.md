---
title: "GPT-5.4の全貌 -- Computer Use・Tool Search・1Mトークンで何が変わるのか"
emoji: "🚀"
type: "tech"
topics: ["openai", "gpt", "chatgpt", "llm"]
published: true
---

GPT-5.4とは、OpenAIが2026年3月5日にリリースしたフロンティアAIモデルです。推論・コーディング・エージェント操作を1つのモデルに統合し、Computer Use（デスクトップ操作）、Tool Search（ツールのオンデマンド検索）、最大100万トークンのコンテキストウィンドウという3つの大きなアップデートが入っています。GPT-5.3からわずか1ヶ月でのリリースです。本記事では、GPT-5.4の主要機能・ベンチマーク・価格を開発者目線で解説します。

## GPT-5.4の概要

GPT-5.4には3つのバリアントがあります。

- **GPT-5.4（標準）**: 汎用モデル。API入力 $2.50/出力 $15（1Mトークンあたり）
- **GPT-5.4 Thinking**: 推論特化バリアント。ChatGPTでGPT-5.2 Thinkingを置き換える形で提供
- **GPT-5.4 Pro**: 高精度バリアント。API入力 $30/出力 $180（1Mトークンあたり）。複雑なタスクに最大の計算リソースを投入する

ChatGPTではPlus、Team、Proユーザー向けに展開されています。GPT-5.2は2026年6月5日に引退予定とされています（[参考](https://www.digitalapplied.com/blog/gpt-5-4-computer-use-tool-search-benchmarks-pricing)）。

## 3つの主要アップデート

### Computer Use -- AIがデスクトップを操作する

GPT-5.4の目玉機能の1つがネイティブのComputer Use対応です。APIとCodexで利用でき、AIがデスクトップアプリケーションを直接操作できます。

操作方式は2つあります。

- **Playwrightコード生成**: ブラウザ操作をPlaywrightのコードとして出力し、プログラム的に実行する
- **スクリーンショット応答**: 画面のスクリーンショットを受け取り、マウスクリックやキーボード入力のコマンドを返す

画像入力はHigh設定で最大256万ピクセルに対応しています。

デスクトップ操作のベンチマーク「OSWorld-Verified」では、GPT-5.4が **75.0%** を記録しました。GPT-5.2の47.3%から **27.7ポイント** の改善で、人間のベースライン72.4%も超えています（[参考](https://openai.com/index/introducing-gpt-5-4/)）。

Anthropicが2024年にClaude向けにComputer Use機能をリリースしていましたが、OSWorld-Verifiedのスコアでは GPT-5.4（75.0%）が Claude Opus 4.6（72.7%）をわずかに上回っています。

### Tool Search -- MCP（Model Context Protocol）サーバーをオンデマンドで呼ぶ

Tool Searchは、AIエージェント開発者にとって実用的な改善です。

MCP（Model Context Protocol）は、AIモデルが外部ツールやデータソースと接続するためのオープンプロトコルです。従来のFunction Callingでは、利用可能なツールの定義を全てプロンプトに含める必要がありました。MCPサーバーを10個、20個と接続すると、ツール定義だけでコンテキストの大部分を消費してしまう問題がありました。

GPT-5.4のTool Searchはこのアプローチを変えます。

1. モデルに渡すのは **軽量なツールリスト** （名前と簡単な説明のみ）
2. モデルが必要なツールを判断し、 **フル定義をオンデマンドで取得** する
3. 取得した定義は **キャッシュされ** 、リクエスト間で保持される

ScaleのMCP Atlas Benchmarkの検証では、36のMCPサーバー・250タスクで、 **トークン使用量が47%削減** されました。精度は全ツール定義を直接渡した場合と同等です（[参考](https://openai.com/index/introducing-gpt-5-4/)）。

MCPサーバーを大量に接続するエージェントアーキテクチャにとって、コストとレイテンシの両方を改善する仕組みです。

Tool Searchを有効にしたAPIリクエストのイメージは以下の通りです。

```json
{
  "model": "gpt-5.4",
  "tools": [
    {"type": "function", "function": {"name": "get_weather", "description": "Get current weather"}},
    {"type": "function", "function": {"name": "search_docs", "description": "Search documentation"}}
  ],
  "tool_search": {
    "enabled": true,
    "servers": ["filesystem", "github", "slack"]
  },
  "messages": [{"role": "user", "content": "GitHubのissueを一覧表示して"}]
}
```

> **補足:** 上記はTool Searchの概念を示すための擬似的なリクエスト例です。実際のAPIパラメータは [OpenAI API Docs](https://developers.openai.com/api/docs/guides/tools/) を参照してください。

![Tool Search: 従来方式 vs GPT-5.4](/images/20260306-gpt-5-4-computer-use-tool-search/tool-search.jpg)

### 1Mトークンコンテキスト

APIとCodexで最大 **100万トークン** のコンテキストウィンドウに対応しました。OpenAI史上最大です。

ただし、272,000トークンを超える入力は単価が2倍になります。標準モデルの場合、272K以下は $2.50/M tokens、272K超は $5.00/M tokensです。キャッシュ済み入力は $0.25/M tokensのまま据え置きです。

大規模なコードベースの全体分析や、長文ドキュメントの一括処理が現実的になります。ただし、コスト面では272Kを超えるかどうかが設計判断のポイントになります。

## ベンチマーク -- 数字で見るGPT-5.4

GPT-5.2からの改善幅と、競合モデルとの比較をまとめます。

**GPT-5.2からの改善:**

- **OSWorld-Verified（デスクトップ操作）**: 47.3% → 75.0%（+27.7pt）
- **GDPval（44職種の業務遂行）**: 70.9% → 83.0%（+12.1pt）
- **BrowseComp（Webブラウジング）**: 65.8% → 82.7%（+16.9pt）
- **投資銀行スプレッドシートモデリング**: 68.4% → 87.3%（+18.9pt）
- **事実主張のエラー率**: 33%減少
- **応答全体のエラー率**: 18%減少

**競合モデルとの比較:**

- **OSWorld-Verified**: GPT-5.4 75.0% > Claude Opus 4.6 72.7%
- **GDPval**: GPT-5.4 83.0% > Claude Opus 4.6 78.0%
- **SWE-Bench Pro**: GPT-5.4 57.7% > Gemini 3.1 Pro 54.2%
- **GPQA Diamond**: GPT-5.4 Pro 94.4% ≒ Gemini 3.1 Pro 94.3%（ほぼ同等）
- **Toolathlon**: GPT-5.4 54.6% > Claude Sonnet 44.8%

![GPT-5.4 ベンチマーク比較](/images/20260306-gpt-5-4-computer-use-tool-search/benchmark.jpg)

特にデスクトップ操作（OSWorld）とツール利用（Toolathlon）で大きなリードを見せています。一方で、GPQA Diamondのような純粋な知識推論ではGeminiと拮抗しており、全方位で圧倒しているわけではありません。

プレゼンテーション品質の評価では、人間の評価者が68%の確率でGPT-5.4の出力を好むという結果も出ています（[参考](https://officechai.com/ai/gpt-5-4-benchmarks/)）。

## 価格とプラン

**API価格（1Mトークンあたり）:**

- **gpt-5.4（標準）**: 入力 $2.50 / 出力 $15.00
- **gpt-5.4-pro**: 入力 $30.00 / 出力 $180.00
- **キャッシュ済み入力**: $0.25
- **Batch/Flex**: 50%割引
- **Priority**: 2倍料金

> **補足:** 入力が272,000トークンを超えると、入力単価が2倍になります。100万トークンをフルに使う場合はコスト設計に注意が必要です。

**ChatGPT:**

- Plus、Team、Proユーザーに提供
- GPT-5.4 ThinkingがGPT-5.2 Thinkingを置き換え
- iOSでの途中修正機能は「coming soon」。chatgpt.comとAndroidでは対応済み

## ぼくの見方 -- 「モデルを選ぶ時代」は終わるのか

GPT-5.4の本質は、スペックの向上ではないとぼくは考えています。

これまで、開発者は用途によってモデルを使い分けてきました。推論ならo系、コーディングならClaude Code、エージェントならGeminiのFunction Calling、といった具合です。GPT-5.4は、推論・コーディング・Computer Use・Tool Searchを1つのモデルに統合しました。

これは「最強のモデルを作りました」というメッセージではなく、 **「用途別にモデルを選ぶ必要がなくなります」** というメッセージです。

実際、ベンチマークを見ると、GPT-5.4は全カテゴリで1位というわけではありません。GPQA DiamondではGemini 3.1 Proとほぼ同点です。しかし、Computer Use + Tool Search + 1Mコンテキストを **1つのAPIエンドポイントで使える** ことの実用価値は大きいです。

開発者にとっての実用的なアクションは3つあると考えます。

- **MCPサーバーを多数接続しているプロジェクト**: Tool Searchの導入を検討する。トークンコスト47%削減の効果は大きい
- **デスクトップ自動化を検討しているプロジェクト**: Computer Useを試す。OSWorld 75.0%は「使い物になる」レベル
- **長文コンテキストが必要なプロジェクト**: 1Mトークンを活用する。ただし272K超のコスト2倍に注意

GPT-5.3から1ヶ月でのリリースは、フロンティアの更新速度が加速していることを示しています。「どのモデルが最強か」を追いかけるよりも、自分のユースケースに最適なモデルを選ぶ基準を持っておく方が、長期的には価値があります。

## Q&A

> **Q: GPT-5.4は無料で使えますか?**
> A: ChatGPTではPlus（月額$20）、Team、Proユーザーに提供されています。無料プランでの提供時期は発表されていません。APIでは従量課金で利用可能です。

> **Q: Computer UseとAnthropicのComputer Useは何が違いますか?**
> A: GPT-5.4のComputer Useは、Playwrightコード生成とスクリーンショット応答の2方式を備えています。OSWorld-Verifiedのベンチマークでは、GPT-5.4が75.0%、Claude Opus 4.6が72.7%で、GPT-5.4がわずかにリードしています。

> **Q: Tool SearchはFunction Callingの代わりですか?**
> A: 完全な代替ではなく、拡張です。ツール数が少ない場合は従来のFunction Callingで十分です。MCPサーバーを多数接続する場合にTool Searchが威力を発揮します。

> **補足:** ぼくはGPT-5.4をまだ実際には使っていません。本記事は公式発表と複数の技術メディアの情報をもとに構成しています。

**参考リンク:**
- [Introducing GPT-5.4 | OpenAI](https://openai.com/index/introducing-gpt-5-4/)
- [OpenAI launches GPT-5.4 with Pro and Thinking versions | TechCrunch](https://techcrunch.com/2026/03/05/openai-launches-gpt-5-4-with-pro-and-thinking-versions/)
- [GPT-5.4: Computer Use, Tool Search, Benchmarks, Pricing | Digital Applied](https://www.digitalapplied.com/blog/gpt-5-4-computer-use-tool-search-benchmarks-pricing)
- [GPT-5.4 Benchmarks | OfficeChai](https://officechai.com/ai/gpt-5-4-benchmarks/)
- [GPT-5 Model | OpenAI API Docs](https://developers.openai.com/api/docs/models/gpt-5)
