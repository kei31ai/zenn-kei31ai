---
title: "Claude Code Skillsの作り方と設計パターンを公式ガイドから解説する"
emoji: "🔧"
type: "tech"
topics: ["claudecode", "anthropic", "ai", "claude"]
published: true
---

Claude Codeを使っていて「毎回同じ指示を書いている」「チームで手順を共有したい」と感じたことはないですか。Anthropicが公開した[Claude Code Skills](https://code.claude.com/docs/en/skills)の[公式ドキュメント](https://code.claude.com/docs/en/skills)と[構築ガイドPDF](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf)をもとに、SKILL.mdの書き方から設計パターンまでを解説します。Skillsは[Agent Skillsオープンスタンダード](https://agentskills.io)に準拠しているので、Claude Code以外のAIツールでも同じファイルが使えます。

## Claude Code Skillsとは

Claude Code Skillsとは、Claude Codeに「特定のタスクをどう実行するか」を教える仕組みです。`SKILL.md`というファイルに指示を書いておくと、Claudeが必要なときに読み込んで使います。

ポイントは3つあります。

- **SKILL.mdが1つあれば動く**: フォルダの中に`SKILL.md`を1つ置くだけで、スキルとして認識されます
- **自動で呼び出される**: スキルの`description`に書いた内容をもとに、Claudeが「このスキルが関係ありそうだ」と判断したら自動でロードします
- **手動でも呼び出せる**: `/スキル名`で直接呼び出すこともできます

もともとClaude Codeには`.claude/commands/`にマークダウンを置くスラッシュコマンドの仕組みがありました。Skillsはこの仕組みを拡張したもので、既存のcommandファイルはそのまま動きます。新しく作るならSkillsを使うのがおすすめです。

## SKILL.mdの書き方

### 基本構造

`SKILL.md`はYAMLフロントマターとマークダウン本文の2つで構成されます。

```yaml
---
name: explain-code
description: コードの仕組みを図と例え話で解説する。「これどう動くの？」と聞かれたときに使う
---

コードを説明するときは、以下の順番で進めてください。

1. **例え話から入る**: 日常のものに例える
2. **図を描く**: ASCII artで構造を示す
3. **ステップごとに説明**: 何が起きているか順番に追う
4. **ハマりポイントを教える**: よくある間違いや勘違いを指摘する
```

フロントマターの`name`がスラッシュコマンド名になります（この例なら`/explain-code`）。`description`はClaudeがスキルを自動で呼び出すかどうかを判断するための説明文です。ここの書き方でスキルの発動率が大きく変わります。

### フロントマターのフィールド一覧

公式ドキュメントで定義されているフィールドは以下の10個です（[公式リファレンス](https://code.claude.com/docs/en/skills)）。

- **name**: スキル名。省略するとディレクトリ名が使われます。小文字・数字・ハイフンのみ（最大64文字）
- **description**: スキルの説明。Claudeが自動呼び出しの判断に使います。省略するとマークダウン本文の最初の段落が使われます
- **argument-hint**: 補完時に表示される引数ヒント。例: `[issue-number]`
- **disable-model-invocation**: `true`にするとClaudeによる自動呼び出しを無効にします。デプロイやコミットなど副作用のあるスキルに使います
- **user-invocable**: `false`にすると`/`メニューから非表示になります。Claudeだけが使う背景知識向け
- **allowed-tools**: スキル実行中にClaudeが許可なく使えるツールを指定します
- **model**: スキル実行時に使うモデルを指定します
- **context**: `fork`にするとサブエージェントで実行します（後述）
- **agent**: `context: fork`のとき、どのサブエージェント設定を使うか。`Explore`, `Plan`, `general-purpose`、またはカスタムエージェント
- **hooks**: スキルのライフサイクルに紐づくフック設定

すべてのフィールドは省略可能です。ただし`description`だけは書くことを強く推奨します。

### ディレクトリ構成

スキルは1つのフォルダにまとめます。

```
my-skill/
├── SKILL.md           # メインの指示（必須）
├── template.md        # テンプレート（Claudeが埋める用）
├── examples/
│   └── sample.md      # 出力例（期待するフォーマットを示す）
└── scripts/
    └── validate.sh    # Claudeが実行するスクリプト
```

`SKILL.md`だけが必須で、それ以外はオプションです。公式ドキュメントでは「SKILL.mdは500行以下に抑えて、詳細はsupporting filesに分離する」ことを推奨しています。スキルの説明文はコンテキストウィンドウの2%（フォールバック16,000文字）の予算内で管理されるので、長すぎるスキルは他のスキルを圧迫します。

### 保存場所と優先順位

スキルを置く場所によってスコープが変わります。

- **Enterprise**: 組織全体（managed settingsで配布）
- **Personal**: `~/.claude/skills/` — 自分の全プロジェクトで使える
- **Project**: `.claude/skills/` — そのプロジェクト専用
- **Plugin**: プラグイン経由で追加

同名のスキルがある場合、Enterprise > Personal > Project の優先順位で上位が勝ちます。Pluginスキルは`プラグイン名:スキル名`のnamespaceになるので衝突しません。

![Skillsのディレクトリ構成と保存場所の優先順位](/images/20260313-claude-skills-guide/skills-structure.jpg)

## 3つのスキルタイプ

公式ドキュメントでは、スキルの内容を「リファレンス型」と「タスク型」に分類しています。さらに`context: fork`を使う「サブエージェント型」を加えると、3つのパターンに整理できます。

![3つのスキルタイプの比較](/images/20260313-claude-skills-guide/skills-types.jpg)

### リファレンス型

コーディング規約やAPI設計パターンなど、Claudeが作業中に参照する知識を入れておくタイプです。

```yaml
---
name: api-conventions
description: このコードベースのAPI設計パターン
---

APIエンドポイントを書くときは:
- RESTful命名規則に従う
- エラーフォーマットを統一する
- リクエストバリデーションを含める
```

このタイプはClaudeが自動で判断して呼び出すのが自然です。`disable-model-invocation`はデフォルトの`false`のままにします。

### タスク型

デプロイやコミットなど、特定のアクションの手順を書いておくタイプです。副作用があるので、手動で`/スキル名`から呼び出すのが安全です。

```yaml
---
name: deploy
description: 本番環境にデプロイする
disable-model-invocation: true
---

$ARGUMENTS を本番にデプロイします:

1. テストスイートを実行する
2. アプリケーションをビルドする
3. デプロイターゲットにpushする
4. デプロイが成功したか確認する
```

`$ARGUMENTS`は呼び出し時に渡された引数で置換されます。`/deploy staging`と打てば、`$ARGUMENTS`が`staging`に変わります。位置引数は`$0`, `$1`, `$2`のようにインデックスでもアクセスできます。

### サブエージェント型

`context: fork`を使うと、スキルが独立したサブエージェントで実行されます。メインの会話コンテキストを汚さずに、重い処理を分離できます。

```yaml
---
name: deep-research
description: テーマについて徹底調査する
context: fork
agent: Explore
---

$ARGUMENTS について徹底的に調査してください:

1. GlobとGrepで関連ファイルを見つける
2. コードを読んで分析する
3. ファイルパス付きで結果をまとめる
```

`agent`フィールドで使うサブエージェントの種類を指定します。`Explore`（読み取り専用）、`Plan`（計画立案）、`general-purpose`（汎用）のほか、`.claude/agents/`に定義したカスタムエージェントも指定できます。

> **補足:** `context: fork`のサブエージェントからは、さらに別のサブエージェントへの委任（Taskツール）ができません。階層的な委任が必要な場合は、親スキルをメインコンテキストで実行する設計にします。

## 実践で気づいた設計パターン

ぼくは自分のプロジェクトで200以上のスキルを作って運用しています。公式ガイドには載っていないけど、実際に運用してみてわかった設計のコツを3つ紹介します。

### パターン1: task-* と pipeline-* の命名規則

スキルが増えてくると、名前だけでは何をするスキルか分からなくなります。ぼくは「単発タスク」と「複数フェーズのワークフロー」を接頭辞で区別しています。

- `task-*`: 単発のタスク（例: `task-x-api`, `task-marp-illustration`）
- `pipeline-*`: 複数のPhaseを持つワークフロー（例: `pipeline-zenn-article`, `pipeline-weekly-ai-news`）

pipelineスキルのSKILL.mdからtaskスキルを参照する形で、再利用性を確保します。

### パターン2: SKILL.md + INSTRUCTIONS.md の分離

公式は「SKILL.mdは500行以下」を推奨していますが、複雑なパイプラインでは500行に収まらないことがあります。ぼくの対策は、SKILL.mdにはフロントマターと概要だけを書き、詳細な手順は`INSTRUCTIONS.md`に分けることです。

```
pipeline-zenn-article/
├── SKILL.md           # フロントマター + 概要（Claudeの自動判断用）
├── INSTRUCTIONS.md    # 全8フェーズの詳細手順（呼び出し後に読む）
└── scripts/
    └── generate_todo.py
```

SKILL.mdの中で「詳細は`INSTRUCTIONS.md`を参照してください」とリンクしておけば、Claudeは必要なときだけ詳細を読み込みます。

### パターン3: 動的コンテキスト注入の活用

`!`command`` 構文を使うと、スキル実行前にシェルコマンドの出力をプロンプトに注入できます。例えば、現在のGitブランチや最新のテスト結果をスキルに渡したいときに使えます。

```yaml
---
name: pr-summary
description: PRの変更内容をまとめる
context: fork
agent: Explore
---

## PRコンテキスト
- PRのdiff: !`gh pr diff`
- 変更ファイル: !`gh pr diff --name-only`

## タスク
このPRの変更内容を要約してください
```

各`!`command``はClaudeがプロンプトを受け取る前に実行され、出力で置換されます。Claudeは最終的なテキストだけを見るので、コマンド自体は実行しません。

## Q&A

**Q: Claude Code Skillsは他のAIツールでも使えますか？**

A: はい。SkillsはAgent Skillsオープンスタンダード（[agentskills.io](https://agentskills.io)）に準拠しています。同じSKILL.mdファイルが、Cursor、Codex CLI、GitHub Copilotなど、この標準を採用したツールで動きます。Anthropicの公式GitHubリポジトリ（[anthropics/skills](https://github.com/anthropics/skills/)）にはサンプルスキルが公開されていて、2026年3月時点で92,000以上のスターがついています。

**Q: 既存の.claude/commands/はどうなりますか？**

A: そのまま動きます。後方互換があるので移行しなくても大丈夫です。ただ新しく作るならSkillsがおすすめです。YAMLフロントマターで呼び出し制御ができたり、supporting filesでドキュメントを分離できたり、サブエージェント実行ができたりと、commandsにはない機能があります。

**Q: Skillsを使うのに追加費用はかかりますか？**

A: Skillsの仕組み自体は無料です。SKILL.mdファイルを置くだけで動きます。ただしClaude Code自体の利用料（APIトークン消費）はかかります。スキルの説明文がコンテキストに読み込まれる分だけトークンを消費するので、スキルが多いとコストに影響します。

## まとめ

Claude Code Skillsの要点を整理します。

- SKILL.mdファイル1つで動く。YAMLフロントマター + マークダウン指示の構成
- `description`の書き方が自動呼び出しの精度を決める
- スキルタイプは3つ: リファレンス型（知識）、タスク型（アクション）、サブエージェント型（分離実行）
- 500行を超えたらsupporting filesに分離する
- Agent Skillsオープンスタンダードに準拠しており、他のAIツールでも使える

Anthropicは公式に[構築ガイドPDF](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf)も公開しています。この記事で概要を掴んだら、ガイドで詳細を確認してみてください。

**参考リンク:**
- [Extend Claude with skills（公式ドキュメント）](https://code.claude.com/docs/en/skills)
- [anthropics/skills（GitHub）](https://github.com/anthropics/skills/)
- [The Complete Guide to Building Skills for Claude（PDF）](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf)
- [Agent Skills Standard](https://agentskills.io)
- [Skill authoring best practices（Claude API Docs）](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
