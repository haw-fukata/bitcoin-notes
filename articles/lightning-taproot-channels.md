---
title: "LightningチャネルをTaproot化すると何が変わるのか"
emoji: "🌳"
type: "tech"
topics: ["bitcoin", "lightning", "taproot"]
published: true
---

## はじめに

LightningチャネルのTaproot化は、Funding Outputのアドレス形式をP2TRへ置き換えるだけの変更ではありません。2者の鍵と署名をMuSig2で集約し、commitment transactionの出力が持つ複数の使用条件をTaprootのKey PathとScript Pathへ割り当てる、チャネル全体に関わる変更です。

この記事では、現在のP2WSHベースのチャネルと比べながら、オンチェーンで何が見え、何が隠れ、どこが小さくなるのかを整理します。

## 先に結論

LightningチャネルをTaproot化すると、主に次の点が変わります。

- Funding OutputをP2TRにし、MuSig2による1つの集約公開鍵で表現できます
- 2者の部分署名を、オンチェーンでは1つのSchnorr署名に集約できます
- 協調クローズを通常のP2TRのKey Path Spendと区別しにくくできます
- commitment transactionの複数の条件をTapTreeへ分け、使わなかった条件を隠せます
- Key Path Spendなどではオンチェーンデータを削減できます
- 強制クローズではタイムロックやトランザクション構造が現れるため、Lightningらしさが完全に消えるわけではありません

この記事でいうTaprootチャネルは、BOLTsへ追加されたextension BOLTの**Simple Taproot Channels**を主な具体例にします。これは段階的な更新であり、PTLCまで同時に導入する仕様ではありません。

2026年7月13日時点で仕様はBOLTsに追加されていますが、対応状況は実装やバージョンによって異なります。extension BOLTも、現段階のチャネルを公開ネットワークへannounceするにはgossip protocolの追加変更が必要だとしています。

## 現在のLightningチャネル

現在一般的なLightningチャネルのFunding Outputは、P2WSH（Pay to Witness Script Hash）の2-of-2マルチシグです。AliceとBobの両方が署名しなければ使用できません。

```text
Aliceの署名 AND Bobの署名
```

BOLT #3が定義するWitness Scriptは、概念的には次の形です。

```text
2 <Aliceの公開鍵> <Bobの公開鍵> 2 OP_CHECKMULTISIG
```

この出力を使用すると、WitnessにはAliceとBobの署名、2つの公開鍵を含むWitness Scriptが現れます。Scriptそのものから2-of-2で共同管理された出力だと分かりやすい構造です。

commitment transactionにも、`to_local`、`to_remote`、offered HTLC、received HTLC、anchor outputなどが含まれます。従来方式では、これらの多くがP2WSHなどのScriptで表現されています。

## Taproot化はP2TRへの置き換えだけではない

Taprootチャネルでは、変更はFunding Outputだけにとどまりません。

| 対象 | 主な変更 |
| --- | --- |
| Funding Output | P2WSH 2-of-2から、MuSig2集約鍵を使うP2TRへ変更 |
| Funding Outputの支出 | 2つのECDSA署名とScriptから、集約したSchnorr署名によるKey Path Spendへ変更 |
| commitment transactionの出力 | `to_local`、`to_remote`、HTLC、anchor outputなどをP2TRで構成 |
| 複数の使用条件 | 1つのScript内の分岐ではなく、複数のTapLeafへ分離 |

P2TR（Pay to Taproot）には、公開鍵に対する署名だけで使う**Key Path**と、Scriptを実行して使う**Script Path**があります。複数のScriptは、各条件をTapLeaf（TapTreeの葉）としてMerkle treeであるTapTreeへまとめます。

Simple Taproot Channelsは、現在のチャネル構造をTaprootへ段階的に移す提案です。したがって「Taprootで一から別のLightningを作る」というより、従来の残高更新やrevocationの意味を保ちながら、オンチェーン表現をP2TR、Schnorr署名、Tapscriptへ移すものと捉えると分かりやすいです。

## MuSig2で鍵と署名を集約する

MuSig2は、複数人が1つの集約公開鍵を作り、その鍵に対する通常のBIP340 Schnorr署名を共同生成するためのn-of-nマルチシグ方式です。2者チャネルでは、AliceとBobの両方が参加してはじめて有効な署名を作れます。

```text
Aliceの公開鍵
Bobの公開鍵
       ↓ MuSig2 KeyAgg
1つの集約公開鍵
```

これは公開鍵を単純に足すだけではありません。BIP327の`KeyAgg`は、公開鍵のリストをハッシュし、各公開鍵に鍵集約係数を適用して集約します。これにより、他者の公開鍵に合わせて悪意ある鍵を選ぶrogue-key attackを防ぎます。

署名時には、両者がnonceを交換して署名セッションを作り、それぞれがpartial signature（部分署名）を生成します。最後に部分署名を集約すると、集約公開鍵で検証できる1つのSchnorr署名になります。

```text
Aliceの部分署名
Bobの部分署名
       ↓ MuSig2 PartialSigAgg
1つのSchnorr署名
```

このためオンチェーンから見ると、複数人の共同署名でも、単独の公開鍵に対する単独の署名に近い形式になります。

## 協調クローズはどう変わるか

従来の協調クローズでは、Funding Outputを使用するために2つの署名と2-of-2のWitness Scriptを公開します。

Taprootチャネルでは、MuSig2でFunding Outputの集約鍵に対する署名を共同生成し、Key Path Spendできます。BIP341でSIGHASHの1バイトを省略する場合、Witnessの中心は64バイトのSchnorr署名1つです。

```text
従来: 2つの署名 + 2-of-2 Witness Script
P2TR: 1つの集約Schnorr署名
```

署名だけを見ても、それが1人で作られたのか、AliceとBobがMuSig2で作ったのかは判別しにくくなります。その結果、次の違いは従来より見分けにくくなります。

- 通常のP2TR出力
- LightningチャネルのFunding Output
- MuSig2を使うその他の複数人署名システム

ただし、これは「Lightningチャネルを完全に識別できない」という意味ではありません。Funding TransactionやClosing Transactionの入出力構成、金額、タイミング、既知のチャネル情報などを組み合わせたヒューリスティックから推測される余地は残ります。

## 強制クローズでは何が起きるか

一方的にチャネルを閉じると、最新状態を表すcommitment transactionがオンチェーンに公開されます。状況に応じて、次のような出力が現れます。

- `to_local`: commitment transactionを公開した側の残高
- `to_remote`: 相手側の残高
- offered HTLC / received HTLC: 未解決の条件付き支払い
- anchor output: CPFPで手数料を追加するための出力

これらには、一定期間待つ、payment preimageを示す、CLTV期限後に回収する、古い状態ならrevocation keyで回収するといった条件があります。

Taprootでは、条件ごとにScriptを別のTapLeafへ分けられます。概念的には次の構造です。

```text
P2TR Output
├── Key Path
└── TapTree
    ├── preimageを使う経路
    ├── timeoutを使う経路
    └── revocationを使う経路
```

ただし、すべての出力がこの3つの葉を持つわけではありません。実際のSimple Taproot Channelsでは出力ごとに構成が異なります。たとえば`to_local`はdelayとrevocationを別のScript Pathにし、HTLC出力はtimeoutとsuccessを別のTapLeafに分けつつ、revocation public keyをinternal keyに使います。つまりHTLCのrevocationはKey Pathで処理でき、追加のScriptを公開せずに済みます。

強制クローズでは、使用されたScript、CSVやCLTVのタイムロック、HTLC解決用の後続トランザクションなどが現れます。したがって、Funding Outputや協調クローズが通常のP2TR支払いに近づいても、強制クローズまで完全に通常の支払いと同じに見えるわけではありません。

## 使用しなかった条件を隠せる

P2WSHを使用すると、支出時にWitness Script全体を公開します。1つのScriptに複数の分岐があれば、実行しなかった条件も観察できます。

TaprootのScript Path Spendでは、主に次のデータを公開します。

- 条件を満たす署名やpreimageなどのWitnessデータ
- 実際に実行するTapscript
- そのTapLeafがTapTreeに含まれることを示すControl Block
- 必要な場合は、Control Blockに含まれるMerkle Path

そのため、同じTapTreeにある別のTapLeafの具体的なScriptは公開せずに済みます。ただしMerkle Pathの長さなどから、隠れた条件の存在や木の深さについて一部推測される可能性はあります。

Key Path Spendでは、基本的に署名だけを公開します。Script Pathの内容だけでなく、そもそもTapTreeがコミットされていたかどうかも、外部からは判断しにくくなります。

## オンチェーンサイズと手数料

P2WSH 2-of-2のFunding Outputを使用するには、2つのECDSA署名とWitness Scriptが必要です。MuSig2によるP2TRのKey Path Spendなら、基本的には1つのSchnorr署名で使用できます。

この差は、特に次のケースでオンチェーンデータの削減につながります。

- Funding Outputを使用するとき
- 協調クローズするとき
- 複数人の署名をMuSig2のKey Pathで処理できる経路

一方、Script Path Spendでは、実行するTapscript、Control Block、Merkle Path、条件を満たすWitnessデータが必要です。木の深さや実行する条件によってサイズは変わるため、Taproot化すればすべての支出が必ず小さくなるとは限りません。

## TaprootチャネルとPTLCは別のもの

Taprootチャネルへ移行しても、HTLCが自動的にPTLC（Point Time Locked Contract）へ置き換わるわけではありません。Simple Taproot Channelsも、まず既存のHTLCベースの構造をTaprootへ移す段階的な更新です。

一方で、BIP340のSchnorr署名を使うチャネルは、Adaptor Signatureを利用する将来のPTLCを組み込みやすくする基盤になり得ます。PTLCの詳しい仕組みは別の記事で扱います。

## 実装は単純になるとは限らない

オンチェーン表現が簡潔になっても、Lightning実装の内部状態まで単純になるとは限りません。MuSig2では、少なくとも次の情報を安全に対応付ける必要があります。

- secret nonceとpublic nonce
- partial signature
- どのcommitment stateを署名しているか
- 再接続時に再開する署名状態
- 古い状態や別のトランザクションとの取り違え防止
- Hardware Signerとの署名セッション

特にsecret nonceは、異なるメッセージや署名セッションで再利用してはいけません。同じsecret nonceを複数の署名へ使うと、partial signatureの関係から秘密鍵が漏れる可能性があります。

Simple Taproot Channelsのextension BOLTは、commitment state、チャネル再接続、協調クローズのRBFなどに合わせたnonce交換と更新方法を定義しています。つまりオンチェーンの1署名を実現するために、オフチェーンではnonceと部分署名を含む追加の状態管理が必要です。

## よくある誤解

### Taproot化すればLightningチャネルを識別できなくなる

協調クローズは通常のP2TR支出と区別しにくくなりますが、トランザクション全体の構造から推測される可能性は残ります。強制クローズではLightning特有のタイムロックや解決トランザクションも現れます。

### TaprootチャネルではPTLCが使われる

TaprootチャネルとPTLCは別の更新です。Taproot化したHTLCチャネルも構成できます。

### MuSig2は公開鍵を足すだけである

鍵集約係数の計算などが必要です。また署名時にはnonce交換、部分署名の検証と集約を行います。

### Taproot化すれば必ず手数料が安くなる

Key Path Spendでは削減を期待できますが、Script Path SpendにはTapscriptやControl Blockなどが必要です。どの経路を使用するかで変わります。

## まとめ

LightningチャネルのTaproot化は、Funding Outputだけでなく、署名方式とcommitment transactionの出力構成を含む変更です。

- Funding OutputをP2TR化できます
- MuSig2で複数の公開鍵と部分署名を1つへ集約できます
- 協調クローズを通常のTaproot支払いと区別しにくくできます
- 使用しなかったScript条件を隠せます
- Key Path Spendなどでオンチェーンデータを削減できます
- 強制クローズ時のLightning特有の構造までは完全に隠せません
- PTLCは別の仕組みであり、詳しくは別の記事で扱います
- nonceや再接続状態の管理など、実装上の複雑さがあります

Taprootチャネルの中心は、「アドレスを変えること」ではなく、協調できる場合は1つの鍵と署名に見せ、協調できない場合は必要な条件だけを公開することです。

## 参考資料

- [BIP340: Schnorr Signatures for secp256k1](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)（2026-07-13確認）
- [BIP341: Taproot](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)（2026-07-13確認）
- [BIP342: Validation of Taproot Scripts](https://github.com/bitcoin/bips/blob/master/bip-0342.mediawiki)（2026-07-13確認）
- [BIP327: MuSig2 for BIP340-compatible Multi-Signatures](https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki)（2026-07-13確認）
- [BOLT #3: Bitcoin Transaction and Script Formats](https://github.com/lightning/bolts/blob/master/03-transactions.md)（2026-07-13確認）
- [Extension BOLT: Simple Taproot Channels](https://github.com/lightning/bolts/blob/master/bolt-simple-taproot.md)（2026-07-13確認）
- [lightning/bolts#995: extension-bolt: simple taproot channels](https://github.com/lightning/bolts/pull/995)（2026-07-13確認）

## 更新履歴

- 2026-07-13: 初版
