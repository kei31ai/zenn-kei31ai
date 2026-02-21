---
title: "スキルを87個運用したら898KBになった。SKILL.md分割で27KBに戻した実測記録"
emoji: "⚡"
type: "tech"
topics: ["claudecode", "ai", "aiagent", "llm"]
published: true
---

:::message
この記事は「Claude Codeで自律型AIエージェントを作る」シリーズ第11回です。
第10回: [Claude Codeのhooksでエージェントの暴走を防ぐ](https://zenn.dev/kei31ai/articles/20260220-claude-code-hooks-behavior-guard)
:::

スキルが増えるほどClaude Codeが遅くなる。原因はSKILL.mdがコンテキストを圧迫しているからだった。SKILL.md（フロントマター+参照1行）とINSTRUCTIONS.md（詳細手順）に分割したところ、87スキル全体で898KB→27KBへ97%削減できた。この記事ではその仕組みと実測データを公開する。

## スキルが増えるほどコンテキストが重くなる

Claude Codeのスキルは便利だ。特定のタスクを呼び出すだけで、複雑な手順をAIが実行してくれる。

しかし、スキルを増やすほど問題が起きる。**スキルが呼ばれるたびに、SKILL.mdの全内容がシステムプロンプトに注入される**のだ。

`task-skill-splitter/SKILL.md`にはこう書かれている。

> 32KB のスキルが3つ呼ばれるだけで 96KB のコンテキストを消費する。

ぼくのプロジェクトは87個のスキルがある。スキルを増やして自動化を進めるほど、コンテキストが膨れ上がり、トークン効率が悪化する。スケールすればするほど自分の首を絞める構造だ。

## 解決策：二段階ロード

解決策はシンプルだ。**SKILL.mdに最小限の情報だけ残し、詳細はINSTRUCTIONS.mdへ分離する。**

**分割前のSKILL.md（全部ひとまとめ）:**

```markdown
---
name: pipeline-zenn-article
description: Full Zenn article writing pipeline...
---

# Zenn記事作成パイプライン

## Phase 1: セットアップ
...（詳細手順）...

## Phase 2: ネタ探し
...（詳細手順）...

## Phase 8: デプロイ
...（詳細手順）...
```
サイズ: **37,813バイト**（常時コンテキストに注入）

**分割後のSKILL.md（フロントマターのみ）:**

```markdown
---
name: pipeline-zenn-article
description: Full Zenn article writing pipeline from trend search to deployment.
---

詳細な手順は `INSTRUCTIONS.md` を参照（このスキルのディレクトリ内）。
```
サイズ: **314バイト**（常時コンテキストに注入されるのはこれだけ）

詳細手順はINSTRUCTIONS.mdへ全移動。Claude Codeはそのスキルの手順が必要なときだけReadツールで読み込む。

二段階ロードの流れはこうなる。

```
スキル呼び出し
    ↓
Stage 1: 軽量なSKILL.mdだけが自動注入（約200〜500バイト）
    ↓
そのスキルの手順が必要になる
    ↓
Stage 2: Readツールで INSTRUCTIONS.md を読む（必要なときだけ）
    ↓
手順に従って実行
```

コンテキストを圧迫するのは「常に注入される」Stage 1だけだ。Stage 2は必要なときだけ読まれる。

![スキルの二段階ロード構造（Before/After比較）](/images/20260221-skill-context-optimization/01-skill-loading-stages.jpg)

## 実測データ：Before / After

`split.py --list` で全87スキルを計測した結果がこれだ。

**代表スキルの削減効果：**

| スキル名 | 分割前（SKILL.md全体） | 分割後（SKILL.mdのみ） | 削減率 |
|---------|----------|-----------------|--------|
| pipeline-zenn-article | 37,813B | 314B | **99.2%** |
| pipeline-autonomous-pdca | 4,796B | 513B | **89.3%** |
| task-nazenaze（未分割） | 1,968B | — | — |

**プロジェクト全体（87スキル）：**

| 状態 | サイズ |
|------|--------|
| 分割前推定（全コンテンツ合計） | **898 KB** |
| 分割後（SKILL.mdのみ、自動注入分） | **27 KB** |
| 削減量 | **871 KB（97%削減）** |

83スキルを分割済み（95.4%）。残り4スキルは未分割のまま（task-nazenaze、task-x-community-post等）。

![87スキルのコンテキスト削減量（実測データ）](/images/20260221-skill-context-optimization/02-skill-context-reduction.jpg)

## 公式ドキュメントの裏付け

Claude Codeの公式ドキュメントでも同じ考え方が示されている。

> Skill descriptions budget: 2% of context window（フォールバック: 16,000文字）
> Keep SKILL.md under 500 lines

スキルの説明（SKILL.md）はコンテキストウィンドウの**2%**が上限として設定されている。87個すべてが「常に注入される」設計なら、それだけでコンテキストの大部分を食い潰してしまう。

公式の推奨（500行以内）を厳格に実装したのが、この二段階ロード構造だ。

## 実装方法

既存のスキルを分割するには `split.py` を使う。

```bash
SCRIPT=.claude/skills/task-skill-splitter/scripts/split.py

# 一覧表示（未分割のスキルを確認）
python3 $SCRIPT --list-unsplit

# ドライラン（プレビュー）
python3 $SCRIPT task-nazenaze --dry-run

# 実行（SKILL.mdを分割してINSTRUCTIONS.mdを生成）
python3 $SCRIPT task-nazenaze

# 5KB以上の全スキルを一括分割
python3 $SCRIPT --all --min-size 5000
```

新規スキルを作るときは、最初から二段階構造にするのがベストプラクティスだ。

```
skills/
└── my-new-skill/
    ├── SKILL.md        # frontmatter + 「詳細はINSTRUCTIONS.mdを参照」のみ
    └── INSTRUCTIONS.md # 詳細な手順・コマンド・ルールを全部書く
```

SKILL.mdは「このスキルは何をするか」だけ。INSTRUCTIONSは「どうやるか」を書く。

## Q&A

**Q: 分割しないとどうなる？**

スキルが増えるほどコンテキストの消費が増え、タスク実行の精度が落ちる可能性がある。ぼくのケースでは37KBのスキル（pipeline-zenn-article）が常にコンテキストに乗っていた。大きなスキルから小さなスキルまで積み重なった結果、87スキル全体で898KBになっていた。

**Q: INSTRUCTIONS.mdはいつ・誰が読む？**

Claude Code（エージェント自身）が「そのスキルの詳細手順が必要」と判断したときにReadツールで読む。スキルを呼び出しただけでは読まれない。必要なときだけロードされる設計だ。

**Q: 未分割のままでいいケースは？**

SKILL.md全体が200〜500バイト程度に収まるなら、分割しなくていい。分割のオーバーヘッド（INSTRUCTIONSを読むタスク）が不要なシンプルなスキルはそのままでOKだ。ぼくのプロジェクトでも4スキルは意図的に未分割のまま残している。

## まとめ

スキルの二段階ロードは「書いておく」と「必要なときに読む」を分ける設計だ。コンテキストに常に乗せるものを最小限にすることで、スキルを増やしても効率を維持できる。実測では87スキル全体で898KB→27KB（97%削減）を達成した。スキルが10個を超えたあたりから、SKILL.md分割を検討することをすすめる。

---

## 参考リンク

- [Claude Code 公式ドキュメント - Skills](https://code.claude.com/docs/en/skills)
- [Claude Code 公式サイト](https://code.claude.com)
- [Anthropic - Claude Code リリース](https://www.anthropic.com/news/claude-code)
- [このシリーズの第1回: Claude Codeで自律型AIエージェントを作る](https://zenn.dev/kei31ai/articles/20260124-claude-code-autonomous-agent)
