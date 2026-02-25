---
title: "Anthropic RSP v3.0の全貌 — AI安全ポリシーが正直になった理由と批判"
emoji: "🛡️"
type: "tech"
topics: ["anthropic", "ai", "llm"]
published: true
---

2026年2月24日、[Anthropic](https://anthropic.com)がResponsible Scaling Policy（RSP）のバージョン3.0を公開しました。AI安全ポリシーの包括的な書き直しです。本記事では、RSP v3.0の変更点、他社フレームワークとの比較、そして外部からの批判を整理します。

## RSPとは何か — AIの「バイオセーフティレベル」

RSP（Responsible Scaling Policy）は、AnthropicがAIモデルの能力向上に伴うリスクに対処するための自主規制フレームワークです。2023年9月に[初版が公開](https://www-cdn.anthropic.com/1adf000c8f675958c2ee23805d91aaade1cd4613/responsible-scaling-policy.pdf)されました。

核となるのは **ASL（AI Safety Level）** という段階制です。バイオセーフティレベル（BSL）に着想を得ており、モデルの能力が特定の閾値を超えたら、より厳格なセーフガードを発動する「if-then」型のコミットメントになっています。

- **ASL-1**: 基礎レベル。重大なリスクを伴わないシステム
- **ASL-2**: 現行の多くのモデルが該当。標準的なセーフガードを適用
- **ASL-3**: 化学・生物兵器リスクなど、特定領域で高度な対策が必要。[2025年5月に発動済み](https://anthropic.com/news/activating-asl3-protections)
- **ASL-4/5**: 国家レベルのリスクに対応。v3.0でも完全な定義はされていない

![AI Safety Levels（ASL）の段階](/images/20260225-anthropic-rsp-v3/asl-levels.jpg)

> **Q: なぜASL-4/5は定義されていないのか?**
> A: AnthropicはASL-4/5の能力を持つモデルがまだ存在しないため、現時点で具体的な基準を設定するのは「推測に基づく」ことになるとしています。能力の進化に合わせて段階的に定義する方針です。

## v3.0で何が変わったのか

v3.0はRSPの「包括的な書き直し」（comprehensive rewrite）です。主な変更点は3つあります。

![RSP v3.0の3つの変更点](/images/20260225-anthropic-rsp-v3/v3-changes.jpg)

### 変更点1 — 「自分たちだけでは無理」の明文化

v3.0の最大の特徴は、 **企業単独コミットメント** と **業界推奨** を明確に分離したことです。

- **企業単独コミットメント**: Anthropicが他社の動向に関係なく、自社で実行する施策
- **業界推奨**: 業界全体で取り組むべき課題。一社で対処するには限界がある領域（例: 国家レベルの攻撃者に対するモデル重みのセキュリティ）

これまでのRSPでは、この区別が曖昧でした。v3.0では「一方的な対応の限界」を公式に認めた上で、自社でやれることとやれないことを分けています。

RANDの報告書でも「モデル重み（モデルの学習済みパラメータ。流出すると第三者がモデルを複製できる）のセキュリティ基準は、現状では達成不可能であり、国家安全保障コミュニティの支援が必要」という見解が示されています（[RSP v3.0本文](https://www-cdn.anthropic.com/e670587677525f28df69b59e5fb4c22cc5461a17.pdf)内で引用）。

### 変更点2 — Frontier Safety Roadmap

新しく導入された[Frontier Safety Roadmap](https://anthropic.com/news/responsible-scaling-policy-v3)は、Security・Alignment・Safeguards・Policyの4領域にわたる具体的な目標を公開するものです。

主な目標には以下が含まれます。

- 情報セキュリティの「ムーンショットR&D」の立ち上げ
- バグバウンティを超えるレッドチーミングの開発
- Claudeの行動に関する「Constitutional」な振る舞い指標の導入
- 内部脅威を検出するための記録システムの構築
- 規制に関する「規制ラダー」政策提案の公表

これらは「拘束力のない（nonbinding）」目標ですが、公開されることで外部からの監視が可能になります。

### 変更点3 — Risk Report + 外部レビュー

v3.0では、 **3〜6ヶ月ごとにRisk Reportを公開** する仕組みが導入されました。

- Risk Reportには、モデルの能力評価、脅威モデル、対策の整合性が含まれる
- **外部の第三者レビューアー** が、編集なし（unredacted）または最小限の編集（minimally-redacted）でアクセスし、Anthropicの判断を公に批評する
- 2025年5月のSafeguards Reportがプロトタイプとして先行公開済み

ガバナンス面では、共同創業者でChief Science Officerの **Jared Kaplan** がResponsible Scaling Officerに就任しています。

## 他社フレームワークとの比較

Anthropic RSP発表後、他の主要AI企業も類似のフレームワークを採用しています。3社のアプローチには明確な違いがあります。

**Anthropic RSP**: バイオセーフティレベルに着想を得た段階制（ASL-1〜5）。能力の閾値を超えたらセーフガードが発動する「if-then」構造

**OpenAI Preparedness Framework**: リスクカテゴリ追跡型。Biological、Cybersecurity、Autonomous Replication、AI Self-Improvementの4領域で、HighとCriticalの2段階の閾値を定義。閾値を超えたモデルは「強力な緩和策なしにデプロイしない」と表明

**Google DeepMind**: deceptive alignment（欺瞞的整列 — AIが意図的に安全そうに振る舞いつつ、内部では別の目標を持つリスク）を独立したリスクとして明示的に識別している点が特徴。Instrumental Reasoning Levelsという独自指標を導入し、モデルが監視を回避したり隠れた目標を追求する能力を評価（[参考](https://www.enkryptai.com/blog/frontier-safety-frameworks-comprehensive-overview)）

> **Q: 3社のフレームワークで最も実効性が高いのはどれか?**
> A: 現時点では比較困難です。3社とも自主規制であり、外部の強制力がありません。ただしAnthropicのv3.0は外部レビュー制度を導入した点で、透明性の面では一歩先を行っています。

## 批判 — 「後退」と指摘する声

v3.0は評価されるだけでなく、批判も受けています。AI安全評価団体[SaferAI](https://www.safer-ai.org/anthropics-responsible-scaling-policy-update-makes-a-step-backwards)は、RSPの評価を2.2から1.9に引き下げました（SaferAI独自の評価基準による。5点満点）。

批判の主な論点は3つです。

**1. 定量から定性への移行**: v1.0では「特定タスクで50%の成功率」のような定量的閾値が定義されていました。v3.0ではこれが「劇的な加速」のような定性的な記述に置き換えられています。SaferAIはこれを「検証可能なコミットメントの後退」と指摘しています。

**2. 「trust us」アプローチ**: モデルが閾値を超えたかどうかをAnthropicが単独で判断する構造は、「ゴールポストが動く」リスクがあるとされています。具体的な安全対策が「目標」レベルの記述に留まり、「何をするか」ではなく「コンプライアンスを証明する」形に変わったことへの懸念です。

**3. 技術の進歩に逆行**: SaferAIは「技術が進歩し、リスク管理が成熟するにつれて、コミットメントはより具体的になるべき」と主張しています。v3.0がその逆方向に進んだことを問題視しています。

## なぜ「限界を認める」ことが重要なのか

批判は正当ですが、ぼくはv3.0の方向性に一定の合理性があると見ています。

理由は2つあります。

**1つ目は、完璧を装うより現実的な設計の方が機能するからです。** v2.0までの定量的閾値は「曖昧ゾーン」の問題を抱えていました。モデルが大半のテストに合格するものの、リスクが低いとも高いとも断定できない状態が発生し、公にリスクを主張する根拠が弱まっていたのです。v3.0が定性的な記述を採用したのは、この現実に対する設計上の判断です。

**2つ目は、外部レビュー制度が「trust us」問題の現実解になり得るからです。** 定量的な閾値があっても、それを評価するのがAnthropic自身なら自己採点と変わりません。v3.0が導入した外部レビュー（unredacted accessでの第三者批評）は、自己判断の限界を構造的に補う仕組みです。

同じ週にAnthropicは[国防総省との$200M契約](https://techcrunch.com/2026/02/24/anthropic-wont-budge-as-pentagon-escalates-ai-dispute/)で自律兵器への使用拒否を堅持しています。安全ポリシーの更新と、実際にポリシーを理由に大型契約を失うリスクを同時に引き受けている点は、ポリシーが飾りでないことの傍証です。

## まとめ

RSP v3.0の変更点を整理すると、以下の通りです。

- 企業単独コミットメントと業界推奨を分離し、「一社でやれる限界」を明文化した
- Frontier Safety Roadmapで具体的な安全目標を公開した
- 3-6ヶ月ごとのRisk Report + 外部第三者レビューを導入した
- 一方で、定量的閾値の後退やコミットメントの曖昧化に対する批判がある

AI安全フレームワークは発展途上です。完璧なポリシーは存在しません。重要なのは、その不完全さを認めた上で改善し続ける仕組みがあるかどうかです。v3.0はその方向に進んでいると、ぼくは判断しています。

**参考リンク:**
- [Responsible Scaling Policy Version 3.0（Anthropic公式）](https://anthropic.com/news/responsible-scaling-policy-v3)
- [RSP v3.0 全文PDF](https://www-cdn.anthropic.com/e670587677525f28df69b59e5fb4c22cc5461a17.pdf)
- [ASL-3 保護の有効化（Anthropic公式）](https://anthropic.com/news/activating-asl3-protections)
- [Anthropic's RSP Update Makes a Step Backwards（SaferAI）](https://www.safer-ai.org/anthropics-responsible-scaling-policy-update-makes-a-step-backwards)
- [Frontier Safety Frameworks 比較（Enkrypt AI）](https://www.enkryptai.com/blog/frontier-safety-frameworks-comprehensive-overview)
- [Anthropic Won't Budge as Pentagon Escalates AI Dispute（TechCrunch）](https://techcrunch.com/2026/02/24/anthropic-wont-budge-as-pentagon-escalates-ai-dispute/)
