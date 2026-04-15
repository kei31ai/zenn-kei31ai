---
title: "Gemma 4が変えたオープンモデルの評価軸：intelligence-per-parameterとは何か"
emoji: "🔥"
type: "tech"
topics: ["gemma", "llm", "生成ai", "google"]
published: true
---

:::message
ぼくはまだGemma 4を実際に動かしていない。本記事は公式ドキュメント・公式ブログ・コミュニティレポートをもとに構成している。実測値には⚠マークで注記する。
:::

Gemma 4（2026年4月2日リリース、Apache 2.0）の登場で、open model競争の評価軸が「総パラメータ数」から「byte for byte capability」へ転換しつつある。31B DenseがArena AIスコア1452でopen model 3位、E2BはRaspberry Pi 5でon-device動作する。4モデルの設計思想と、この転換が持つ意味を整理する。

---

## 評価軸が変わった

少し前まで、open modelの「強さ」はパラメータ数で語られることが多かった。

Llama 4 Maverick は128エキスパートを持つMoEモデルで、総パラメータ数は400Bを超える。それでもArena AIスコアは1417。一方、Gemma 4 31B Denseは名前の通り31Bのdenseモデルで、スコアは1452。10倍以上のパラメータ差を逆転した。

Google DeepMindはGemma 4のコアメッセージを「byte for byte, the most capable open models」と表現した。これが「intelligence-per-parameter」という評価軸への転換を象徴している。

**intelligence-per-parameter** とは「同じパラメータ数（≒推論コスト・メモリ消費）あたりでどれだけの知能を発揮できるか」という尺度だ。モデルが大きいから賢いのではなく、コンパクトなサイズで高い能力を実現できるかが問われる。

---

## Gemma 4とは — 4モデルの設計思想

Gemma 4は2026年4月2日にGoogle DeepMindがリリースした。全モデルApache 2.0ライセンスで商用利用が可能。4サイズ展開で、用途ごとに明確な役割分離がされている。

| モデル | 有効パラメータ | コンテキスト | 主な用途 |
|--------|--------------|------------|---------|
| E2B | 2.3B（埋め込み含む5.1B） | 128K | Android・Pi・Jetson Nano |
| E4B | 4.5B | 128K | エッジ高性能 |
| 26B MoE | 3.8B（アクティブ）/ 25.2B（総量） | 256K | クラウド軽量推論 |
| 31B Dense | 31B | 256K | フロンティアopen model |

E2BとE4Bは「E」がEdge（エッジ）を意味する。E2Bはハイブリッドアテンション（ローカルスライディングウィンドウ512トークン+グローバルアテンション）とPer-Layer Embeddings（PLE）を採用した設計で、モバイルデバイス・シングルボードコンピュータでの動作を前提にしている。

26B MoEは総パラメータ25.2Bを持ちながら、推論時にアクティブになるのは3.8Bのみ。MoE（Mixture of Experts）構造で、実際の計算コストをE4B相当に抑えながら大モデルの知識を活用できる。

全モデルがテキスト・画像・音声・動画をネイティブに処理するマルチモーダル対応。140以上の言語をサポートする。

![Gemma 4 — 4モデルラインナップ比較](/images/20260415-gemma4-open-model-intelligence/fig1-model-lineup.jpg)

---

## ベンチマークで見る実力

31B Denseのベンチマーク（公式）:

- **Arena AI**: 1452（open model 3位）
- **MMLU Multilingual**: 85.2%
- **AIME2026**: 89.2%
- **LiveCodeBench**: 80.0%

AIME2026 89.2%は数学推論の高さを示しており、LLMの苦手分野とされてきた領域での向上が顕著だ。LiveCodeBench 80.0%はコード生成能力で、実用的な開発ワークフローへの組み込みを想定したスコアと見てよい。

140言語対応のMMLU Multilingual 85.2%は多言語処理の安定性を示す。日本語での実際のパフォーマンスは別途検証が必要だが、多言語対応の基盤としての水準は高い。

![open model Arena AIスコア比較](/images/20260415-gemma4-open-model-intelligence/fig2-arena-score.jpg)

---

## エッジデバイスでの動作

Gemma 4の特徴的な訴求点のひとつが、エッジデバイスでの実動作だ。

**Raspberry Pi 5 + LiteRT-LM**

Google公式ブログによると、LiteRT-LMを使用したRaspberry Pi 5での計測値:
- prefill速度: **133トークン/秒**
- 4000入力トークンを **3秒未満** で処理

コミュニティの実測値（⚠条件不明）では、生成速度は8〜12トークン/秒程度とのレポートがある。prefill速度と生成速度は測定方法が異なるため、単純比較はできない。Pi 5での実用的な会話応答の目安として「8〜12トークン/秒」を参考値として押さえておくとよい。実際に動かす場合は量子化（4-bit等）の有無によって速度・メモリ消費が大きく変わるため、環境に合わせた設定確認が必要だ。

**Android + NPUアクセラレーション**

AndroidではMediaPipe/LiteRTによってNPU（Neural Processing Unit）アクセラレーションが有効になる。ML Kit GenAI Prompt APIとの組み合わせで、アプリ開発者がGemma 4をon-device推論に利用できる。

さらにAgent Skillsと呼ばれる機能がon-deviceで動作する: Wikipedia検索、動画要約、他モデルとの連携。これらがすべてデバイス上で完結するのは、agentic workflowsのプライバシー要件を満たすうえで意義が大きい。

---

## Apache 2.0の意味

Gemma 3は独自ライセンスを採用していた。downstream利用への制約があり、ライセンス条件をGoogleが一方的に変更できる条項が含まれていた（⚠二次ソースのみ）。企業がGemma 3を本番環境に組み込む際のリスク要因になっていた。

Gemma 4でApache 2.0に転換したことで:

- **商用利用の制限なし**: 改変・再配布・商用利用がすべて自由
- **ライセンス安定性**: Apacheは確立された標準ライセンスで、条件が一方的に変わることがない
- **GDPR/データ主権との親和性**: on-premise運用が容易になり、クラウド依存を避けたい組織が採用しやすい

open modelの「open」がどこまでを指すかは常に議論になるが、Apache 2.0採用はエコシステムへのコミットメントとして明確なシグナルだ。

---

## 実際にどう使うか

Gemma 4は主要なMLフレームワーク・ツールで動作する。

**Ollama**
```bash
# E2B（最軽量）
ollama run gemma4:2b

# 31B Dense
ollama run gemma4:27b
```

**vLLM / LangChain**
大規模バッチ処理や本番API構築にはvLLMが有効。LangChainとの統合でRAGやagent構成も組みやすい。

**Hugging Face Transformers**

```python
from transformers import AutoTokenizer, AutoModelForCausalLM
import torch

tokenizer = AutoTokenizer.from_pretrained("google/gemma-4-E2B")
model = AutoModelForCausalLM.from_pretrained(
    "google/gemma-4-E2B",
    torch_dtype=torch.bfloat16,
    device_map="auto",
)
inputs = tokenizer("Gemma 4の特徴を教えてください", return_tensors="pt").to(model.device)
outputs = model.generate(**inputs, max_new_tokens=200)
print(tokenizer.decode(outputs[0]))
```

function callingは6つの特殊トークンで実装されており、structured JSON outputとあわせてagentic workflowsへの組み込みが標準化されている。

---

## Q&A

**Q: E2BとE4Bはどう使い分ける？**

E2Bは速度優先。有効パラメータ2.3Bで最も軽く、Android・Raspberry Pi・Jetson Nanoでの動作が想定されている。E4Bは同じエッジ向けで有効パラメータ4.5Bと少し大きく、精度を重視したいシーンに向く。どちらも128Kコンテキスト対応。「とにかく軽く動かしたい」ならE2B、「エッジでも精度を落としたくない」ならE4Bが出発点になる。

**Q: Gemma 4は日本語に強い？**

140言語対応でMMLU Multilingual 85.2%を達成している。多言語基盤としての水準は高いが、日本語特有のタスク（敬語の生成、長文要約など）での実力は実機検証が必要。公式ベンチマークに日本語単体のスコアは現時点では見当たらない。

---

## まとめ

Gemma 4が示したのは、「パラメータ数の競争」ではなく「byte for byte capability」という新しい評価軸だ。31B DenseがLlama 4 Maverick（400B）を超えるArena AIスコアを記録し、E2BはRaspberry Pi 5でon-device agentとして動作する。Apache 2.0への転換は商用利用・データ主権の両面で企業採用のハードルを下げた。

open model競争は「大きいモデルを作れるか」から「同じコストでどこまで賢くできるか」へ軸足が移っている。Gemma 4はその転換を最もはっきり体現したモデルだと思う。

---

## 参考リンク

- [Google Blog: Gemma 4: Byte for byte, the most capable open models](https://blog.google/innovation-and-ai/technology/developers-tools/gemma-4/)
- [Google DeepMind: Gemma 4 モデルページ](https://deepmind.google/models/gemma/gemma-4/)
- [Hugging Face: google/gemma-4-E2B](https://huggingface.co/google/gemma-4-E2B)
- [Google Developers Blog: Bring agentic skills to the edge with Gemma 4](https://developers.googleblog.com/bring-state-of-the-art-agentic-skills-to-the-edge-with-gemma-4/)
- [Android Developers Blog: Gemma 4 new standard for local agentic intelligence](https://android-developers.googleblog.com/2026/04/gemma-4-new-standard-for-local-agentic-intelligence.html)
