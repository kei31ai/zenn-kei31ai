---
title: "Claude Code Voice Modeとは — 音声でコードを書く新しい開発体験"
emoji: "🎙️"
type: "tech"
topics: ["claudecode", "anthropic", "ai", "CLI"]
published: true
---

2026年3月3日、AnthropicはCLIツール「[Claude Code](https://docs.anthropic.com/en/docs/claude-code)」にVoice Modeを追加しました。`/voice`コマンドで有効化し、スペースバーを押しながら話すだけでコードの実装を指示できます。本記事では、Voice Modeの機能・使い方・制限事項を整理し、音声入力がコーディング体験をどう変えるかを解説します。

> **補足:** 執筆時点（2026年3月4日）でVoice Modeは約5%のユーザーに限定公開中。ぼく自身は未アクセスのため、公式情報と外部ソースをもとに書いています。

## Claude Code Voice Modeとは

Claude Code Voice Modeは、AnthropicのCLIコーディングツール「Claude Code」に追加された音声入力機能です。Anthropicのエンジニア[Thariq Shihipar氏がX上で発表](https://x.com/trq212/status/2028628570692890800)しました。

従来のClaude Codeはテキスト入力のみでしたが、Voice Modeによって音声での指示が可能になりました。ターミナル上で`/voice`と入力するだけで有効化でき、有料プラン（Pro、Max、Team、Enterprise）のユーザーは追加費用なしで利用できます。

ポイントは、これが単なる「音声入力」ではなく、Claude Codeのエージェント機能と統合されている点です。「認証ミドルウェアをリファクタリングして」と話しかけるだけで、Claude Codeがファイルの読み込み・編集・テスト実行まで自律的に行います。キーボードを打つ必要がないため、コードを目で追いながら「ここ、何が起きてる？」と聞くような使い方ができます。

## 使い方と基本操作

### 有効化

```bash
# Claude Codeのセッション内で
/voice
```

ウェルカムスクリーンにVoice Modeへのアクセス通知が表示されたユーザーのみ利用可能です。現時点では約5%に限定されており、数週間かけて順次拡大される予定です。

### 基本操作

![Voice Mode操作フロー](/images/20260304-claude-code-voice-mode/flow.jpg)

- **スペースバーを長押し** → 話す → **離す** → 送信
- テキスト入力と音声入力はシームレスに切り替え可能

[Claude Help Center](https://support.claude.com/en/articles/11101966-using-voice-mode)によると、Voice Modeにはhands-freeモード（自動で聞き取り）とpush-to-talkモード（ボタンを押している間だけ聞き取り）の2種類があります。CLIでのコーディング中は、push-to-talk（スペースバー長押し）がメインです。

### ボイス設定

複数のボイスオプションが用意されており、Settings内のVoice settingsから選択できます。なお、一部の報道ではElevenLabsのTTS技術が使われているとされていますが、Anthropicは音声合成エンジンの詳細を公式には開示していません。

## 音声コーディングで何が変わるか

Voice Modeの本質は「音声入力がついた」ことではなく、**開発者の思考モードが変わる**ことにあります。

![タイピングvs音声の思考モード比較](/images/20260304-claude-code-voice-mode/compare.jpg)

### タイピングと音声で指示の粒度が変わる

キーボードで入力するとき、開発者は「何を実装するか」を書こうとします。

```
ユーザー認証のミドルウェアを追加して。JWTトークンの検証をして、
期限切れの場合は401を返す。
```

これに対して、音声で話すとき、人はもっと自然な言葉を使います。

```
ユーザー認証の部分なんだけど、いまセキュリティ的に心配なところがあって。
ログインしたあとのトークン管理、ちゃんとできてるか確認したいんだよね。
```

タイピングは「実装指示」になりやすく、音声は「設計の相談」になりやすい。入力方法が変わるだけで、AIとのやり取りの質が変わります。

### ハンズフリーの実用性

実際のコーディングでは「コードを読みながら考える」場面が多くあります。画面を見ながらキーボードから手を離してAIに質問できるのは、コードレビューやデバッグ時に有効です。

例えば、エラーのスタックトレースを目で追いながら「この3行目のNullPointerException、何が原因だと思う？」と聞く。従来ならスタックトレースをコピーしてプロンプトに貼り付ける手順が必要でしたが、Voice Modeではその手間がなくなります。

ペアプログラミングで同僚に「ここ、もっとシンプルにできない？」と話しかける感覚に近い。設計は声で相談し、実装の細部はキーボードで——そういう使い分けが自然にできます。

## 現在の制限と注意点

### 言語

ベータ機能であり、対応言語は**英語のみ**です。日本語での音声入力は現時点ではサポートされていません。

### ロールアウト状況

[9to5Mac](https://9to5mac.com/2026/03/03/anthropic-adding-voice-mode-to-claude-code-in-gradual-rollout/)によると、2026年3月3日時点で**約5%のユーザー**に限定公開中です。数週間かけて順次拡大される予定ですが、具体的なスケジュールは公開されていません。

### 利用制限

音声での会話は通常のusage limitsにカウントされます。追加の制限も無料枠もありません。音声文字起こし（STT）トークンが無料という報道もありますが、Anthropic公式では未確認です。

### 環境制約

- 静かな環境でないと認識精度が下がる
- 機密コードを音声で話すプライバシー上の懸念
- 変数名・パス名など精密な構文はタイピングの方が正確

## 競合との比較

Claude Code Voice Modeは、AIコーディングツールの音声対応としては後発です。

- **Codex CLI + Wispr Flow**: [ScreenApp](https://screenapp.io/blog/claude-code-voice-mode)等の報道によると、OpenAIのCodex CLIは2026年2月下旬にWispr Flowとの統合で音声入力に対応。外部ツールのインストールが必要です
- **VoiceMode MCP**: Claude Code向けのサードパーティMCPサーバー（[getvoicemode.com](https://getvoicemode.com/)）。非公式ソリューションです

Claude Code Voice Modeの特徴は、**CLI内にネイティブ統合**されている点。`/voice`コマンドひとつで使い始められます。

## Q&A

**Q: Voice Modeを使うのに追加料金はかかりますか？**

A: かかりません。Pro、Max、Team、Enterpriseプランに含まれています。

**Q: 日本語で使えますか？**

A: 現時点では英語のみです。日本語対応の時期は未発表です。

**Q: いつ全ユーザーに開放されますか？**

A: 具体的な日程は公開されていません。Anthropicは「数週間かけてランプアップする」としています。

**Q: 音声入力だけで完結しますか？**

A: テキストと音声をシームレスに切り替えて使えます。精密な入力はキーボード、大まかな指示は音声、という使い分けが想定されています。

**参考リンク:**
- [Thariq Shihipar氏の発表ポスト（X）](https://x.com/trq212/status/2028628570692890800)
- [Using voice mode - Claude Help Center](https://support.claude.com/en/articles/11101966-using-voice-mode)
- [Claude Code rolls out a voice mode capability - TechCrunch](https://techcrunch.com/2026/03/03/claude-code-rolls-out-a-voice-mode-capability/)
- [9to5Mac - Anthropic adding voice mode to Claude Code](https://9to5mac.com/2026/03/03/anthropic-adding-voice-mode-to-claude-code-in-gradual-rollout/)
