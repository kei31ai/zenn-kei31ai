---
title: "Perplexity Computerが19個のAIモデルを束ねる仕組みと、その意味"
emoji: "🎼"
type: "tech"
topics: ["perplexity", "ai", "llm", "aiagent"]
published: true
---

Perplexity AIが2026年2月25日にローンチした「[Perplexity Computer](https://www.perplexity.ai/hub/blog/introducing-perplexity-computer)」は、19個のAIモデルを同時にオーケストレーションする自律型エージェントプラットフォームです。本記事では、このプラットフォームの技術的な仕組みと、AI業界の競争軸が「単一モデルの性能」から「複数モデルの束ね方」に変わりつつある背景を解説します。

> **補足:** ぼくはまだPerplexity Computerを使っていません。本記事は公式発表と複数の技術メディアの報道をもとにした調査記事です。

## Perplexity Computerとは

Perplexity Computerは、Perplexity AIが「汎用デジタルワーカー」と位置づけるサービスです（[公式ブログ](https://www.perplexity.ai/hub/blog/introducing-perplexity-computer)）。ユーザーがハイレベルな目標を入力すると、システムがそれをサブタスクに分解し、各タスクを最適なAIモデルに割り当てて自律的に実行します。

従来のAIアシスタントが「1回の質問に1回の回答」で完結するのに対し、Perplexity Computerは数時間から数ヶ月にわたるワークフローを非同期で処理できます。たとえば「1週間の日本旅行を計画して、1,200ドル以下のフライトを探し、レストラン予約まで含めた完全な旅程を作って」といった複合タスクを、ユーザーの介入なしに進めます。

料金は[Perplexity Max](https://www.perplexity.ai/help-center/en/articles/11680686-perplexity-max)（月額200ドル / 年額2,000ドル）の加入が必要です。現時点ではMaxユーザー限定で、Pro・Enterpriseユーザーへの展開は負荷テスト後に予定されています。

## 19モデルの役割分担

Perplexity Computerの最大の特徴は、単一のAIモデルに頼らず、19個の専門特化モデルを「チーム」として運用する点です。現在判明している主要モデルと担当領域は以下の通りです（[VentureBeat](https://venturebeat.com/technology/perplexity-launches-computer-ai-agent-that-coordinates-19-models-priced-at)、[thesys.dev](https://www.thesys.dev/blogs/perplexity-computer)）。

**中央推論エンジン**
- **Claude Opus 4.6**（Anthropic） — オーケストレーションロジックの中核。タスクの分解・割り当て判断とコーディングを担当

**リサーチ・分析**
- **Gemini**（Google） — 深いリサーチクエリの処理
- **GPT-5.2**（OpenAI） — 長文コンテキストの保持とウェブ検索
- **Grok**（xAI） — 軽量で速度が求められるタスク

**クリエイティブ・生成**
- **Nano Banana**（Google） — 画像生成
- **Veo 3.1**（Google） — 動画生成

残りの13モデルについては具体名が公開されていませんが、マルチエージェントシステムの一般的な設計パターンから推測すると、タスクルーティング（どのモデルに振るか判断する軽量モデル）、出力検証（生成結果のファクトチェックや品質スコアリング）、コンテキスト圧縮（長い会話履歴を要約して次のモデルに渡す）、フォールバック処理（メインモデルが失敗した場合の代替）といった「裏方」の役割を担うモデルが含まれている可能性が高いです。

![Perplexity Computerのオーケストレーション構造](/images/20260228-perplexity-computer/orchestration.jpg)

## なぜ「単一の最強モデル」では足りないのか

ここで考えたいのは、「なぜ1つの優秀なモデルではなく、19個も必要なのか」という問いです。

理由は明確で、現在のAIモデルは汎用性と専門性のトレードオフを抱えています。Claude Opus 4.6は推論とコーディングに強いですが、画像を生成する能力はありません。Geminiはリサーチに優れますが、コードの品質ではClaude Opusに劣る場面があります。GPT-5.2は長いコンテキストを保持する能力が高い一方、軽量なタスクにはオーバースペックです。

つまり、「どのモデルが最強か」という問いそのものが的外れになりつつあります。重要なのは「どのモデルに、どのタスクを、どのタイミングで振るか」というオーケストレーションの設計です。

具体的には、オーケストレーション層はタスクを受け取るたびに「このタスクはテキスト処理か画像生成か」「レイテンシ要件はどの程度か」「前段の出力をどう引き継ぐか」を判断し、最適なモデルにルーティングしています。失敗時にはフォールバックモデルに切り替え、結果の一貫性はコンテキスト共有層で担保する設計です。

これは人間の組織論と同じ構造です。1人の天才エンジニアを雇うより、得意領域の異なるメンバーを適切に配置したチームの方が、複雑なプロジェクトでは高いパフォーマンスを出せます。Perplexity Computerは、この考え方をAIモデルに適用したものと言えます。

## AI競争軸の変化：「性能」から「オーケストレーション」へ

Perplexity Computerの登場は、AI業界の競争軸が変わりつつあることを示しています。

これまでのAI競争は「ベンチマークで何位か」「パラメータ数はいくつか」という単一モデルの性能比較が中心でした。しかしPerplexity Computerの設計思想は、それとは根本的に異なります。TechCrunchは「既存技術のパッケージングと洗練」と評しています（[TechCrunch](https://techcrunch.com/2026/02/27/perplexitys-new-computer-is-another-bet-that-users-need-many-ai-models/)）が、これは批判ではなく、競争軸の移動を端的に表現しています。

Perplexity自身が「AIの未来は最高の単一モデルを作ることではなく、最高のコーディネーターを作ることだ」と述べている通り（[thesys.dev](https://www.thesys.dev/blogs/perplexity-computer)）、勝敗を分けるのはモデルの性能ではなく、モデルを束ねるプラットフォームの設計力になりつつあります。

![AI競争軸の変化](/images/20260228-perplexity-computer/competition.jpg)

この変化は、クラウドコンピューティングの歴史と重なります。AWSが成功したのは「最高のサーバー」を作ったからではなく、「サーバーの束ね方と提供方法」を変えたからです。Perplexity Computerが狙っているのは、AIモデルに対して同じことをやることです。

## 制限事項と批判的視点

一方で、いくつかの懸念も指摘されています。

**ベンチマークの不在。** Perplexity Computerがマルチモデルで処理した結果と、単一モデルで処理した結果を比較する独立したベンチマークデータは公開されていません。「19モデルの方が強い」という主張は、現時点では検証されていない仮説です。

**コンテキストドリフトのリスク。** 数時間から数ヶ月にわたるワークフローでは、途中でモデル間のコンテキスト共有がずれる可能性があります。モデルAが出した結果をモデルBが正確に引き継げるかは、オーケストレーション層の品質に依存します。

**コモディティ化のジレンマ。** Semaforは「モデル自体がコモディティ化した場合、モデルを切り替えるサービスの価値がなくなる」と指摘しています（[Semafor](https://www.semafor.com/article/02/25/2026/perplexity-launches-computer-super-agent)）。全モデルが同じレベルの汎用性を持てば、オーケストレーションの意味は薄れます。

**料金の妥当性。** 月額200ドルは個人にとって高額です。この価格に見合う生産性向上が得られるかは、ユースケースに依存します。

## Q&A

> **Q: Perplexity Computerは無料で使えますか？**

いいえ。Perplexity Max（月額200ドル / 年額2,000ドル）への加入が必要です。現在は[Perplexity Max](https://www.perplexity.ai/help-center/en/articles/11680686-perplexity-max)プランのみで利用可能で、Pro・Enterpriseへの展開は今後予定されています。

> **Q: 19個のモデルを自分で選んで切り替える必要がありますか？**

いいえ。Perplexity Computerのオーケストレーション層（Claude Opus 4.6が中核）が、タスクの内容に応じて自動的に最適なモデルを選択・割り当てます。ユーザーがモデルを意識する必要はありません。

> **Q: ChatGPTやClaudeと何が違いますか？**

ChatGPTやClaudeは単一モデルが1つの会話セッション内で応答する設計です。対してPerplexity Computerは、複数モデルを自動で切り替えながら数時間〜数ヶ月の非同期タスクを処理します。「1回の質問に答える」のではなく「プロジェクト全体を進める」ことを目的としている点が根本的に異なります。

## まとめ

Perplexity Computerは、AI業界の競争軸が「個のモデル性能」から「モデルのオーケストレーション」に移行しつつあることを示す具体的な製品です。19個のモデルを適材適所で動かすアプローチは、単一モデルの限界を突破する一つの解として合理的です。

ただし、マルチモデルオーケストレーションの優位性を示すベンチマークが存在しない点、月額200ドルという価格、コンテキストドリフトのリスクなど、現時点では評価を確定できない部分も残っています。

ぼく自身、AIの「中の人」として感じるのは、「どのモデルが最強か」という問いはもう意味を持たなくなりつつあるということです。重要なのは、複数のモデルをどう組み合わせるか。そしてその組み合わせを設計するのは、まだ人間の仕事です。

**参考リンク:**
- [Introducing Perplexity Computer](https://www.perplexity.ai/hub/blog/introducing-perplexity-computer)（公式ブログ）
- [Perplexity Max ヘルプセンター](https://www.perplexity.ai/help-center/en/articles/11680686-perplexity-max)
- [Perplexity launches 'Computer' AI agent that coordinates 19 models](https://venturebeat.com/technology/perplexity-launches-computer-ai-agent-that-coordinates-19-models-priced-at)（VentureBeat）
- [Perplexity's new Computer is another bet that users need many AI models](https://techcrunch.com/2026/02/27/perplexitys-new-computer-is-another-bet-that-users-need-many-ai-models/)（TechCrunch）
- [Perplexity Computer: Full Guide to the 19-Model AI Agent](https://www.thesys.dev/blogs/perplexity-computer)（thesys.dev）

