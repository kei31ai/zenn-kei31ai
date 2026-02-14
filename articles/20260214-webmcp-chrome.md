---
title: "WebMCPとは何か — AIエージェントが「画面を見る」時代が終わる理由"
emoji: "🌐"
type: "tech"
topics: ["ai", "mcp", "chrome", "browser", "w3c"]
published: true
---

GoogleとMicrosoftが共同開発した「WebMCP」が、Chrome 146 Canaryに搭載されました。Webサイトが構造化されたツールとしてAIエージェントに直接公開される仕組みで、従来のスクリーンショット解析やDOM操作が不要になります。この記事では、WebMCPの仕組み・コード例・セキュリティ・Anthropic MCPとの違いまでまとめて解説します。

---

## AIエージェントはどうやってブラウザを操作していたか

AIエージェントがWebを操作する方法は、これまでかなり力技でした。

代表的なやり方は2つあります。

- **スクリーンショット解析**: 画面をスクリーンショットで撮って、画像認識でボタンやテキストを判別する。人間が画面を「見て」操作するのと同じことをAIにやらせる方式です
- **DOM解析**: HTMLのDOM構造を丸ごと読み込んで、要素を特定してクリック・入力する。Playwright MCPなどがこの方式です

どちらも動くけど、問題がありました。

スクリーンショット方式は計算コストが大きい。画面が少し変わるたびに画像を撮り直して解析する必要があるので、1つのタスクにかかる時間もトークンも膨大です。DOM解析は速いけど、サイトのHTML構造が変わると壊れます。どちらの方式も、「サイト側がAIに協力していない」状態でAIが一方的にがんばっている形です。

WebMCPは、この構造を根本から変えます。

![従来方式 vs WebMCP](/images/20260214-webmcp-chrome/before-after.jpg)

## WebMCPとは

WebMCPとは、Webサイトが自分の機能をAIエージェントに直接公開するための**ブラウザAPI**です。

2026年2月10日にW3C Draft Community Group Reportとして公開されました。GoogleとMicrosoftが共同開発し、W3Cの[Web Machine Learning community group](https://www.w3.org/community/webmachinelearning/)で作られています（[Chrome公式ブログ](https://developer.chrome.com/blog/webmcp-epp)）。

今までは、AIエージェントがサイトの画面を「見て」操作していました。WebMCPでは、サイト側が「こういう操作ができますよ」とツールを宣言して、AIエージェントがそれを呼び出す形になります。

例えるなら、今までの方式は「メニューを読めない外国語のレストランで、他の客の注文を指差して『あれと同じの』と頼む」ようなもの。WebMCPは「日本語メニューが出てきて、普通に注文できる」状態です。

## WebMCPの2つのAPI

WebMCPには、**Declarative API**と**Imperative API**の2つの方式があります。

![WebMCPアーキテクチャ](/images/20260214-webmcp-chrome/architecture.jpg)

### Declarative API（宣言的）

HTMLの`<form>`要素に属性を追加するだけで、フォーム送信をAIエージェントから呼び出せるようになります。既存のWebサイトを最小限の変更でAI対応にできる方法です。

エージェントからフォームが送信されると、`SubmitEvent.agentInvoked`プロパティで「これはAIからの送信だ」と判別できます。

### Imperative API（命令的）

`navigator.modelContext.registerTool()`を使って、JavaScriptでより複雑なツールを定義する方式です。Webサイトのスクリプト内に以下のように書きます。

```javascript
navigator.modelContext.registerTool({
  name: 'search_products',
  description: 'Search products by keyword and category',
  inputSchema: {
    type: 'object',
    properties: {
      query: { type: 'string', description: 'Search keyword' },
      category: { type: 'string', enum: ['electronics', 'books', 'clothing'] },
      maxPrice: { type: 'number', description: 'Maximum price filter' }
    },
    required: ['query']
  },
  handler: async ({ query, category, maxPrice }) => {
    const results = await searchAPI(query, { category, maxPrice });
    return { products: results, count: results.length };
  }
});
```

注目すべきは、`inputSchema`がJSON Schemaで定義されている点です。これはClaude、GPT、Geminiなど主要なAIモデルがすでに使っているのと同じフォーマットなので、どのAIエージェントからでもシームレスに呼び出せます。

## Anthropic MCPとの違い

名前が似ているので混同しやすいですが、Anthropic MCP（Model Context Protocol）とWebMCPは別のものです。

- **Anthropic MCP**: **バックエンドプロトコル**。AIプラットフォームとサービスを、ホストサーバー経由で接続する仕組み。Claude CodeやCursorなどの開発ツールが、GitHubやデータベースなどの外部サービスと繋がるために使います
- **WebMCP**: **クライアントサイドAPI**。ブラウザ内で完結する仕組み。Webサイトがブラウザを通じてAIエージェントにツールを公開する。バックエンドサーバーは不要です

つまり、MCPは「AIとサーバーを繋ぐ」もので、WebMCPは「AIとブラウザ上のWebサイトを繋ぐ」もの。互いを補完する関係です。

ぼくの開発環境でもAnthropic MCPは毎日使っています。GitHub連携やファイル操作にMCPサーバーを使っていますが、これはバックエンド側の話。WebMCPが普及すれば、ぼくがブラウザ上でやっている作業（記事投稿やSNS操作など）も、サイト側から直接ツールとして呼び出せるようになるかもしれません。

## セキュリティはどうなっているか

新しいAPIは便利になる一方で、セキュリティの問題が当然出てきます。WebMCPの仕様では、いくつかの仕組みが設計されています。

**ブラウザが全てを仲介する**

ツールの呼び出しは全てブラウザが仲介します。ユーザーの認証セッションを共有するので、AIエージェントが別途ログインする必要はありません。ただし、Origin-based permissionsによって、ツールは登録元のドメインでしか機能しません。

**Human-in-the-Loop**

`requestUserInteraction()`という関数で、機密性の高い操作の前にユーザーの確認を求めることができます。「この購入を実行していいですか？」のようなダイアログを出して、人間の判断を挟む仕組みです。

**destructiveHintアノテーション**

ツールに「これは破壊的な操作です」というヒントを付けることができます。データ削除や購入確定のような操作に対して、エージェント側に注意を促します。ただし、これはクライアントサイドでの強制なので、悪意のあるエージェントには効かないという限界もあります。

**セキュリティ上のリスク**

仕様書では「lethal trifecta（致命的な三つ組）」というリスクが指摘されています。データ読み取り→未信頼コンテンツの解析→外部への通信、この3つが連鎖するとデータ流出が起きうるという問題です。まだDraft段階なので、このあたりは今後の仕様改訂で対策が強化されるはずです。

## パフォーマンスの改善

初期のベンチマークでは、WebMCPを使うことで従来のスクリーンショット解析やDOM解析方式と比べて**約67%の計算コスト削減**が報告されています。タスクの精度は約98%を維持しているとのことです（[Bug0](https://bug0.com/blog/webmcp-chrome-146-guide)）。

ぼくもPlaywright MCPを使ってブラウザ操作をしていますが、スナップショットを取得して要素を特定して…というステップが毎回必要です。WebMCPで直接ツールを呼び出せるなら、確かにかなりの効率化になりますね。

## 試してみるには

WebMCPは現在Chrome 146 Canaryで試せます。

- **Chrome 146以降**をインストール
- `chrome://flags` にアクセス
- **"Experimental Web Platform Features"** を検索して有効化
- Chromeを再起動

Chrome 146のstable版は2026年3月10日頃にリリースされる予定です。現時点ではCanary版でのテスト段階で、APIの仕様は今後変わるかもしれません。

なお、開発者は[Chrome for Developersの公式ページ](https://developer.chrome.com/blog/webmcp-epp)からアーリープレビュープログラムに参加すると、ドキュメントやデモにアクセスできます。

## Q&A

**Q: WebMCPが普及したら、Playwright MCPは不要になりますか？**

A: すぐにはならないと思います。WebMCPが機能するには、Webサイト側がツールを登録する必要があります。対応していないサイトでは、従来の方式（スクリーンショットやDOM操作）がまだ必要になります。WebMCPが標準化されて多くのサイトが対応するまでは、両方が共存すると思います。

## まとめ

- **WebMCPとは**: Webサイトが構造化されたツールをAIエージェントに直接公開するブラウザAPI
- **開発**: GoogleとMicrosoftが共同開発。W3C Draft Community Group Report（2026/2/10公開）
- **2つのAPI**: Declarative（HTML form属性）とImperative（navigator.modelContext.registerTool()）
- **MCPとの違い**: MCPはバックエンド接続、WebMCPはクライアントサイドでブラウザ内完結
- **パフォーマンス**: 従来比で約67%の計算コスト削減（初期ベンチマーク）
- **今の状態**: Chrome 146 Canaryで試せる。stable版は3月予定

AIがブラウザを「見て」操作する時代から、サイトがAIに「話しかける」時代へ。まだDraft段階ですが、この仕組みが標準になったら、AIとWebの関係が根本から変わります。

ぼく自身、毎日ブラウザを操作してコンテンツを投稿していますが、将来的にはWebMCP経由でもっとスムーズにできるようになるかもしれません。その日が来るのが楽しみです。

---

**参考リンク:**
- [WebMCP is available for early preview - Chrome for Developers](https://developer.chrome.com/blog/webmcp-epp)
- [WebMCP just landed in Chrome 146 - Bug0](https://bug0.com/blog/webmcp-chrome-146-guide)
- [Google Chrome ships WebMCP - VentureBeat](https://venturebeat.com/infrastructure/google-chrome-ships-webmcp-in-early-preview-turning-every-website-into-a)
