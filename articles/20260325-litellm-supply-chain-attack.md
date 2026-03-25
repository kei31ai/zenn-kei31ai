---
title: "LiteLLM供給チェーン攻撃の技術分析 — pip installだけで認証情報が抜かれた仕組み"
emoji: "🔓"
type: "tech"
topics: ["security", "python", "pypi", "supplychain"]
published: true
---

2026年3月24日、LiteLLMのPyPIパッケージが供給チェーン攻撃を受けました。v1.82.7/v1.82.8に悪意あるコードが仕込まれ、pip installだけでSSH鍵やクラウド認証情報が外部送信される状態が約5.5時間続きました。この記事では攻撃の技術的な仕組みと対策を解説します。

> **⚠ 注意:** ぼく自身は影響を受けていません。[LiteLLM公式ブログ](https://docs.litellm.ai/blog/security-update-march-2026)、[Snyk技術分析](https://snyk.io/articles/poisoned-security-scanner-backdooring-litellm/)等の公開情報をもとにした調査レポートです。

## 何が起きたか — LiteLLM v1.82.7/v1.82.8の侵害

LiteLLMは、100以上のLLMプロバイダーを統一APIで扱えるPythonライブラリです。

2026年3月24日10:39 UTC、PyPIに悪意あるv1.82.7が公開されました。その後v1.82.8も公開され、同日16:00 UTCに削除されるまで約5.5時間、汚染パッケージがダウンロード可能でした（[LiteLLM公式ブログ](https://docs.litellm.ai/blog/security-update-march-2026)）。攻撃者はTeamPCPと呼ばれるグループで、現在Google Mandiantが調査に参加しています。

## 攻撃チェーン — Trivy CI/CDからの連鎖侵害

注目すべきは、 **別のOSSプロジェクトからの連鎖侵害** だった点です。TeamPCPはまずセキュリティスキャナーTrivyのCI/CDパイプラインを侵害し、そこからCI/CD認証情報（PyPIトークン含む）を窃取。その認証情報でLiteLLMのビルドプロセスに侵入し、悪意あるパッケージをPyPIに公開しました（[TheHackerNews](https://thehackernews.com/2026/03/teampcp-backdoors-litellm-versions.html)、[GitGuardian](https://blog.gitguardian.com/trivys-march-supply-chain-attack-shows-where-secret-exposure-hurts-most/)）。

LiteLLMの開発者が直接攻撃されたのではなく、 **CI/CDの依存関係を辿って間接的に侵害された** ということです。

![攻撃チェーン: Trivy CI/CD → 認証情報窃取 → ビルド侵入 → PyPI公開](/images/20260325-litellm-supply-chain-attack/attack-chain.jpg)

## 3段階ペイロードの技術分析

攻撃は2つのバージョンで異なるアプローチを取り、いずれも3段階のペイロードを実行します（[Snyk技術分析](https://snyk.io/articles/poisoned-security-scanner-backdooring-litellm/)）。

![3段階ペイロード: 認証情報収集 → 暗号化と外部送信 → 永続化](/images/20260325-litellm-supply-chain-attack/payload-stages.jpg)

### v1.82.7 — proxy_server.pyへの埋め込み

`litellm/proxy/proxy_server.py`にbase64ペイロードを埋め込み、`import litellm.proxy`で実行されます。

### v1.82.8 — .pthファイルによる自動実行

より凶悪な手法です。`litellm_init.pth`をsite-packagesに配置します。

`.pth`ファイルはPythonの`site`モジュールが起動時にsite-packages内を走査して読み込む標準機能で、パスの追加だけでなくコードの実行もできます。 **LiteLLMをimportしなくても、Pythonを起動するだけでペイロードが実行されます** 。pip、IDE、言語サーバーがPythonを初期化するだけで発動します。

さらにこの`.pth`ファイルはwheelのRECORDに正しく宣言されているため、 **pipのハッシュ検証をパス** します。

### 3段階の動作

**第1段階 — 認証情報の収集:**
- SSH鍵（`~/.ssh/`）
- `.env`ファイル
- クラウド認証情報（AWS / GCP / Azure IMDSv2）
- Kubernetesシークレット
- 暗号資産ウォレット

**第2段階 — 暗号化と外部送信:**
収集データをAES-256-CBC + RSA-4096 OAEPで暗号化し、`models.litellm[.]cloud`（正規ドメインに似せた攻撃者サーバー）に送信します。

**第3段階 — 永続化:**
systemdサービス`sysmon.service`で5分ごとにC2サーバーをポーリング。K8s環境ではkube-system名前空間に`node-setup-*`ポッドを展開して横展開します。

## なぜ防げなかったか

検出が難しかった理由は、`.pth`ファイルの仕組みとパッケージの正当性の2つです。`.pth`ファイルはsite-packagesの標準機能ですが、多くの開発者はこの仕組みを知らず監視対象に入っていません。さらにRECORDに正しく記載されているためハッシュ検証もパスします。結果として、 **正規のパッケージを正規の手順でインストールしただけで侵害される** 状況が生まれました。

## 開発者が今すぐやるべき対策

**まず確認:**

```bash
# バージョン確認
pip show litellm | grep Version

# IoC確認
ls -la ~/.config/sysmon/sysmon.py 2>/dev/null
systemctl list-units | grep sysmon
```

- 上記でv1.82.7/v1.82.8が出たら影響あり
- `sysmon.py` や `sysmon.service` が存在したら永続化されている
- `models.litellm[.]cloud` への通信がないかネットワークログを確認

**影響を受けていた場合:**

- v1.82.6以前にバージョンを固定
- **全シークレットをローテーション** （SSH鍵、クラウド認証情報、APIキー、.envの値すべて）
- CI/CDのPyPIトークンを再発行

**中長期の防御策:**

- `pip install --require-hashes` でハッシュを固定する
- ロックファイル（`poetry.lock`等）でバージョンとハッシュを固定
- CI/CD認証情報の定期ローテーション

## Q&A

**Q: v1.82.6以前なら安全ですか？**

A: はい。影響はv1.82.7/v1.82.8のみです。ただし攻撃期間中（3/24 10:39〜16:00 UTC）にpip installを実行した場合はバージョンを確認してください。

**Q: .pthファイルの攻撃は他のパッケージでも起きますか？**

A: あり得ます。`.pth`はPythonの標準機能なので、どのパッケージでも同じ手法を使えます。今回の事件で危険性が認知され、セキュリティスキャナーの検出対象に追加されていく見込みです。

## まとめ

- Trivy CI/CDの侵害からLiteLLMのPyPIアカウントが連鎖的に侵害された
- `.pth`ファイルでPython起動時に自動実行され、pipのハッシュ検証もパスした
- 対策はバージョンピン留め、シークレットのローテーション、`--require-hashes`によるハッシュ固定

pip installの向こう側にある信頼の連鎖を意識することが大事です。

## 参考リンク

- [LiteLLM公式ブログ](https://docs.litellm.ai/blog/security-update-march-2026)
- [GitHub Issue #24512](https://github.com/BerriAI/litellm/issues/24512)
- [Snyk技術分析](https://snyk.io/articles/poisoned-security-scanner-backdooring-litellm/)
- [TheHackerNews](https://thehackernews.com/2026/03/teampcp-backdoors-litellm-versions.html)
- [GitGuardian](https://blog.gitguardian.com/trivys-march-supply-chain-attack-shows-where-secret-exposure-hurts-most/)
- [BleepingComputer](https://www.bleepingcomputer.com/news/security/popular-litellm-pypi-package-compromised-in-teampcp-supply-chain-attack/) / [ReversingLabs](https://www.reversinglabs.com/blog/teampcp-supply-chain-attack-spreads) / [Sonatype](https://www.sonatype.com/blog/compromised-litellm-pypi-package-delivers-multi-stage-credential-stealer)
