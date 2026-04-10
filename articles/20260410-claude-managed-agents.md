---
title: "Claude Managed Agents完全解説: agent runtimeがmanagedになった意味と使い方"
emoji: "🤖"
type: "tech"
topics: ["claude", "anthropic", "aiagent", "生成ai", "llm"]
published: true
---

:::message
**未検証記事について**: Claude Managed AgentsはPublic Beta（waitlist制）のため、本記事は公式ドキュメント・公式ブログ・Anthropicエンジニアリングブログをもとに執筆しています。実機での動作確認はできていません。コード例は公式Quickstartドキュメントからの引用です。
:::

2026年4月8日、Anthropicは「Claude Managed Agents」をPublic Betaとして発表した。これは新しいモデルでも、プロンプト改善でもない。**agent runtimeそのものがmanaged cloudサービスになった**という、AIエージェント開発のアーキテクチャが変わる転換点だ。本記事では、何が変わったのか・なぜ重要なのか・どう使うのかを体系的に解説する。

## agent runtimeのセルフホストはなぜ辛いのか

AIエージェントを本番環境に投入するとき、「モデルを呼び出す」部分はAnthropicのAPIが担う。しかし問題はその周辺にある。

コードを1行も書く前に、以下を自前で用意しなければならない:

- **Sandboxing**: エージェントがBashを実行するとき、ホスト環境を壊さない隔離環境
- **Checkpointing**: 長時間タスク中にネットワークが切れても状態を保持する仕組み
- **Credential management**: エージェントに渡すAPIキーや権限の安全な管理
- **Permissions**: エージェントが「何をしていいか」のスコープ制御
- **Observability**: ツール呼び出しの追跡、失敗モードの可視化

あるレポートによれば、これらのインフラ整備に従来の開発チームは**数ヶ月単位**を費やしていた。エージェントロジックを書く前の準備だけで、だ。

Claude Managed Agentsはこの問題に対して、「その責任ごとAnthropicが引き受ける」という答えを出した。

## Claude Managed Agentsとは何か

一言で言えば、**agent runtimeのmanaged cloud化**だ。

従来の構成:
```
開発者のコード → Anthropic API（モデル呼び出し）
            ↑
     自前インフラ（sandbox/checkpoint/permissions）
```

Claude Managed Agents後の構成:
```
開発者のコード → Claude Managed Agents API（runtime込み）
                   ↑
          Anthropicが管理（sandbox/checkpoint/permissions）
```

開発者はエージェントロジックだけに集中できる。インフラの再発明が不要になる。

なお、これは**既存のMessages APIとは別の製品**だ。エンドポイントが異なり、Beta headerも必要となる。

## アーキテクチャ: 4つの核概念

![Claude Managed Agentsのアーキテクチャ（Agent/Environment/Session/Eventsの4概念の関係）](/images/20260410-claude-managed-agents/arch.jpg)

Claude Managed Agentsは4つのオブジェクトで構成される。

### Agent

ツール・パーミッション・使用モデルを定義した設定オブジェクト。永続化されるため、一度作れば繰り返し使える。

```python
agent = client.agents.create(
    name="my-agent",
    model="claude-opus-4-6-20251001",
    tools=[{"type": "agent_toolset_20260401"}]
)
```

### Environment

Agentが動作するサンドボックス実行環境。隔離されたLinuxコンテナとして提供される。標準ツールセット（`agent_toolset_20260401`）には以下が含まれる:

- Bash実行
- ファイル操作
- Web検索
- MCP連携

### Session

Agentの1回の実行インスタンス。状態はAnthropicが管理し、**ネットワーク切断後もcheckpointingで継続**する。数時間にわたる複雑なタスクでも、接続が途切れて最初からやり直しということがなくなる。

### Events（SSEストリーム）

セッション実行中のイベントをServer-Sent Events（SSE）でリアルタイムに受信する。主なイベント種別:

| イベント | 説明 |
|---------|------|
| `agent.message` | モデルのテキスト出力 |
| `agent.tool_use` | ツール実行（Bash等） |
| `session.status_idle` | セッション待機状態 |

Anthropicのエンジニアリングブログでは、このアーキテクチャを **"brain vs hands"** と表現している。Claudeが推論層（brain）、Sessionが実行コンテナ（hands）。両者を分離することで、推論の品質とインフラの安定性を独立して最適化できる。

## APIの基本的な使い方

全エンドポイントで以下のBeta headerが必要だ:

```
anthropic-beta: managed-agents-2026-04-01
```

SDKを使う場合は自動で設定される。

基本フローはシンプルに3ステップ:

```python
import anthropic

client = anthropic.Anthropic()

# 1. Agentを作成
agent = client.agents.create(
    name="research-agent",
    model="claude-opus-4-6-20251001",
    tools=[{"type": "agent_toolset_20260401"}]
)

# 2. Sessionを開始
session = client.sessions.create(
    agent_id=agent.id
)

# 3. SSEでイベントを受信
for event in client.sessions.events(session.id):
    if event.type == "agent.message":
        print(event.content)
    elif event.type == "agent.tool_use":
        print(f"Tool: {event.tool_name}")
```

主要エンドポイント:

| エンドポイント | 用途 |
|--------------|------|
| `POST /v1/agents` | Agent作成 |
| `POST /v1/environments` | Environment作成 |
| `POST /v1/sessions` | Session開始 |
| `GET /v1/sessions/{id}/events` | SSEストリーム |

## 料金の計算方法

![Sessionのライフサイクルと課金タイミング（running時のみ課金、idle時は無料）](/images/20260410-claude-managed-agents/lifecycle.jpg)

料金体系は**2層構造**だ。

**Session runtime料金**: $0.08/session-hour
- activeな `running` 状態の時間のみ課金
- idle状態（`session.status_idle`）は**無料**
- ミリ秒単位の精度で計測

**トークン料金**: 使用モデルの標準料金
- Opus 4.6: input $15/M tokens、output $75/M tokens
- Sessions APIで発生したトークンも通常通り課金

具体例（Opus 4.6、1時間のセッション、50K input + 15K output）:
```
Session runtime: $0.08
Input tokens:    50,000 / 1,000,000 × $15 = $0.75
Output tokens:   15,000 / 1,000,000 × $75 = $1.125
合計:            $1.955
```

※この計算は公式ドキュメントに掲載されている仮定値に基づく。実際のコストはタスクの種類・モデル・active時間比率によって異なる。

なお、セッション内でWeb検索を使う場合は別途 $10/1,000回 が加算される。

## セルフホストとの比較

Claude Managed Agentsが向いているケース:
- エンタープライズで早期デプロイが優先される
- observabilityを自前で構築したくない
- 長時間・複数ステップのタスクが多い

セルフホスト（LangGraph、OpenAI Agents SDK等）が向いているケース:
- マルチモデル対応が必要
- データを自社インフラ外に出せない（エアギャップ要件）
- Amazon Bedrock / Google Vertex AI経由での利用が前提
- ベンダーロックインを避けたい

重要な制約として、Claude Managed Agentsは**Claudeモデルのみ対応**。他のモデルに切り替えたい場合、オーケストレーション層の再構築が必要になる。また、Amazon BedrockやGoogle Vertex AI経由では現時点で提供されていない（二次ソース情報）。

## Q&A

**Q: LangGraphやOpenAI Agents SDKと何が違う？**

LangGraph等はオーケストレーションフレームワークで、インフラは自分で用意する必要がある。Claude Managed Agentsはsandboxing・checkpointing・observabilityをAPIに含んでいるため、フレームワーク+インフラのセットを提供しているイメージに近い。ただしモデルはClaude固定。

**Q: idleタイムは料金がかかる？**

かからない。`session.status_idle`（エージェントが次の指示を待っている状態）は課金対象外。activeな実行時間のみ$0.08/hourが発生する。

**Q: マルチエージェント連携はできる？**

現時点では「research preview」段階で、別途アクセス申請が必要。GA（一般提供）ではない。self-evaluation（自己評価）機能も同じくresearch preview。

## まとめ

Claude Managed Agentsは「新しいClaudeモデル」ではない。**agent runtimeをAnthropicが引き受けるというアーキテクチャの転換**だ。

sandboxing・checkpointing・permissionsを自前で実装してきた開発者にとって、この変化は「ツールが増えた」ではなく「構造が変わった」に近い。エージェントインフラの再発明を止めて、エージェントロジックに集中できる環境が整備された。

現時点はPublic Beta（waitlist制）。一部のプレビュー機能（multi-agent、self-evaluation）は別途申請が必要だが、基本機能は今すぐ試せる。

## 参考リンク

**公式ドキュメント**

- [Claude Managed Agentsブログ発表](https://claude.com/blog/claude-managed-agents)
- [Claude Managed Agents Overview（公式ドキュメント）](https://platform.claude.com/docs/en/managed-agents/overview)
- [Claude Managed Agents Quickstart（公式）](https://platform.claude.com/docs/en/managed-agents/quickstart)
- [料金ページ（公式）](https://platform.claude.com/docs/en/about-claude/pricing)
- [Anthropic Engineering Blog: Scaling Managed Agents](https://www.anthropic.com/engineering/managed-agents)

**参考記事（二次ソース）**

- [With Claude Managed Agents, Anthropic wants to run your AI agents for you（The New Stack）](https://thenewstack.io/with-claude-managed-agents-anthropic-wants-to-run-your-ai-agents-for-you/)
- [Claude Managed Agents: What It Actually Offers（Medium）](https://medium.com/@unicodeveloper/claude-managed-agents-what-it-actually-offers-the-honest-pros-and-cons-and-how-to-run-agents-52369e5cff14)
