---
title: "CISAとは何か — BIP458・459・460で理解する入力間の署名集約"
emoji: "🧩"
type: "tech"
topics: ["bitcoin", "bip", "schnorr"]
published: true
---

![同じトランザクションの入力3個について、通常の署名192バイト、半集約128バイト、完全集約64バイトを比較。数値は署名データのみ。](/images/bitcoin-cisa-signature-aggregation/overview.png)
*小さくなるのは署名データです。使うUTXOや送金先の情報まで消えるわけではありません。*

Bitcoinでは、複数のUTXOをまとめて使うと、入力ごとに署名が必要になります。**CISA（Cross-Input Signature Aggregation）は、同じトランザクション内の複数入力の署名をまとめ、データ量を減らす仕組みです。**

2026年7月に提出されたBIP459・460の草案PRでは、その具体的な方式が議論されています。10月5日に確認したPRはOpenで、以下は有効化済みの機能ではなく、提案の解説です。[BIP459のPR](https://github.com/bitcoin/bips/pull/2210)・[BIP460のPR](https://github.com/bitcoin/bips/pull/2212)

## 何をまとめるのか

UTXOは、まだ使われていないトランザクション出力です。3つのUTXOを使うなら、トランザクションには3つの入力が並びます。

通常のTaprootの鍵経路による支出では、各入力に基本64バイトのSchnorr署名を付けます。入力3個なら署名本体は192バイト、10個なら640バイトです。署名がどの取引情報を保証するかを指定する`sighash`によっては、追加の1バイトが必要です。

CISAでは、各入力を使う権限があることを、集約した署名で検証します。**入力の参照先や公開鍵ごとの検証条件は残し、署名の表現を小さくします。** [BIP460草案](https://raw.githubusercontent.com/fjahr/bips/cisa/bip-0460.mediawiki)

## 3つのBIPの役割

| 提案 | 定めること | 署名データの大きさ |
| --- | --- | --- |
| BIP458 | 作成済みの署名を半集約する方式 | `32 × (n + 1)`バイト |
| BIP459 | 対話しながら完全集約署名を作る方式 | 64バイト |
| BIP460 | それらをBitcoinの入力検証に組み込むルール | 上記に制御情報などを追加 |

`n`は集約する入力数です。458・459が署名方式、460がコンセンサスへの組み込みを担当します。署名方式が定まるだけでは、Bitcoin上でその形式を使えるようにはなりません。[BIP460の設計と依存仕様](https://raw.githubusercontent.com/fjahr/bips/cisa/bip-0460.mediawiki)

### 半集約：署名してからまとめる

Schnorr署名は、大まかには32バイトのnonceに由来する成分と、32バイトの数値成分からなります。半集約では前者を署名ごとに残し、後者を共通の32バイトへまとめます。

```text
3個の署名：64 × 3 = 192バイト
半集約後：32 × 3 + 32 = 128バイト
```

署名者同士で追加のnonce交換をせず、それぞれが作った署名を後から集約できるのが利点です。入力数が増えると、署名データはおよそ半分に近づきます。[BIP458草案](https://raw.githubusercontent.com/fjahr/bips/halfagg/bip-0458.mediawiki)

ただし、BIP460では集約への参加を含めたメッセージに署名します。既存のTaproot署名を任意に拾って、そのままCISAに置き換えられるという意味ではありません。[BIP460のSigning](https://raw.githubusercontent.com/fjahr/bips/cisa/bip-0460.mediawiki)

### 完全集約：対話して64バイトを作る

BIP459は、DahLIASという方式を仕様化する草案です。署名者が公開nonceを交換し、その結果を使って部分署名を作り、最後に64バイトの集約署名を得ます。

半集約のように完成済みの通常署名を後処理するのではなく、最初から共同の署名手順を実行します。やり取りはコーディネーターで仲介できますが、署名者側で必要な検証を行い、秘密鍵は渡しません。

削減量は大きい一方、参加者の離脱やセッションのやり直しへの対応が必要です。特に**秘密nonceを別の署名セッションで再利用すると、秘密鍵の漏えいにつながります。** nonceの生成と使い捨ては、実装上の重要な条件です。[BIP459草案](https://raw.githubusercontent.com/fjahr/bips/fullagg/bip-0459.mediawiki)

## MuSig2と何が違うのか

MuSig2は、複数の鍵の持ち主が**同じメッセージ**へ共同署名し、集約公開鍵で検証できる通常のSchnorr署名を作る方式です。たとえば、1つのLightningチャネルの資金を2者で管理するときに使えます。[BIP327](https://raw.githubusercontent.com/bitcoin/bips/master/bip-0327.mediawiki)

CISAがまとめるのは、**異なる入力に対応する公開鍵と署名対象の組**です。同じトランザクションでも、入力位置などによって署名対象のメッセージは異なります。入力をまたぐこの違いを扱うため、MuSig2を使うだけではCISAになりません。[BIP459の署名・検証仕様](https://raw.githubusercontent.com/fjahr/bips/fullagg/bip-0459.mediawiki)

## BIP460で何を変えるのか

確認した草案は、新しい**Witness version 2**の出力を導入し、その鍵経路による支出を集約対象にします。現在のTaproot出力はWitness version 1なので、自動的に対象にはなりません。スクリプト経路内の署名も集約対象外です。

各入力は半集約・完全集約・非集約を選べます。集約署名のデータは入力のWitnessへ分配され、各集約グループの最後の入力に、方式を示す1バイトのマーカーを置きます。集約方式も署名対象に含めることで、第三者が非集約の署名を勝手に集約することを防ぎます。

非集約を残す理由の一つは、特定の署名がオンチェーンに現れることを利用する、一部のAdaptor Signatureプロトコルとの互換性です。[BIP460のWitness・署名メッセージ・Security](https://raw.githubusercontent.com/fjahr/bips/cisa/bip-0460.mediawiki)

## どれくらい軽くなるのか

入力10個を1グループにまとめる場合、署名本体だけなら次の比較になります。

| 方式 | 署名データ |
| --- | ---: |
| 通常 | 640バイト |
| 半集約 | 352バイト |
| 完全集約 | 64バイト |

ただし、マーカー、Witnessの個数・長さを表す情報、必要に応じたsighash指定などは別です。入力・出力も残るため、**「署名が10分の1だから、手数料も10分の1」とはなりません。** 手数料への効果はトランザクション全体のweightと、その時点の手数料率によります。

恩恵が大きい用途は、多数のUTXOの整理や、CoinJoin・PayJoinなど複数人で作る取引です。一方で、完全集約には対話と状態管理が増え、導入には合意形成とウォレット対応も必要になります。[BIP460の用途とWitness構造](https://raw.githubusercontent.com/fjahr/bips/cisa/bip-0460.mediawiki)

## 参考資料

確認日：2026-10-05。提案者の`cisa`／`fullagg`ブランチの草案を対象にしています。仕様は変更される可能性があります。表のサイズは式による比較で、ノードを動かした実測ではありません。

- [BIP460草案：CISA for Taproot Key Path Spends in SegWit v2](https://raw.githubusercontent.com/fjahr/bips/cisa/bip-0460.mediawiki)
- [BIP458草案：半集約署名](https://raw.githubusercontent.com/fjahr/bips/halfagg/bip-0458.mediawiki)
- [BIPs #2212：BIP460の議論](https://github.com/bitcoin/bips/pull/2212)
- [BIP459草案：DahLIASによる完全集約署名](https://raw.githubusercontent.com/fjahr/bips/fullagg/bip-0459.mediawiki)
- [BIPs #2210：BIP459の議論](https://github.com/bitcoin/bips/pull/2210)
- [BIP327：MuSig2](https://raw.githubusercontent.com/bitcoin/bips/master/bip-0327.mediawiki)

## 更新履歴

- 2026-10-05: 初版
