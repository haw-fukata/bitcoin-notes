---
title: "LightningのChannel Jamming Attackとは何か — HTLCを詰まらせるDoS攻撃を理解する"
emoji: "🚧"
type: "tech"
topics: ["bitcoin", "lightning", "security"]
published: true
---

## はじめに

Lightning Networkでは、支払いをルーティングするためにチャネル上の流動性とHTLC枠を使います。これらは有限です。

**Channel Jamming Attack**は、この有限リソースを意図的に占有し、他のユーザーの支払いを通りにくくするDoS攻撃です。

この記事では、HTLCの制約とhodl invoiceの考え方を前提に、Channel Jamming Attackがなぜ成立するのかを説明します。

この連載では、次の順番でLightningのHTLC保留とjammingを見ていきます。

1. [HTLCの制約](/articles/lightning-htlc-constraints)
2. [hodl invoiceとは何か](/articles/lightning-hodl-invoice)
3. Channel Jamming Attack

## 先に結論

Channel Jamming Attackの要点は、次のとおりです。

- 攻撃者は未解決HTLCを意図的に残します
- 狙うリソースは主にチャネルの流動性とHTLC枠です
- 大きな金額で流動性を拘束する攻撃をliquidity jammingと呼びます
- 小額HTLCを大量に作ってHTLC枠を埋める攻撃をHTLC jammingと呼びます
- 支払いを成功させなければ、中継手数料を払わずに済みやすい点が問題です
- 中継ノードは、下流側HTLCが安全に片付く前に上流側HTLCを雑にfailできません
- 対策は、手数料設計、プライバシー、正当な失敗支払いの扱いが絡むため難しいです

Channel Jammingは、ノードを直接停止させる攻撃ではありません。支払い経路を詰まらせる攻撃です。

## 前提：HTLCは有限リソースである

Lightningの支払いでは、経路上にHTLCが作られます。HTLCは、preimageが公開されれば成功し、期限まで公開されなければ失敗する条件付き支払いです。

中継ノードでは、incoming HTLCとoutgoing HTLCが対応しています。

```text
送金者 A
  |
  | incoming HTLC
  v
中継ノード B
  |
  | outgoing HTLC
  v
受取人 C
```

HTLCには次のような制約があります。

- 未解決HTLC数の上限
- 未解決HTLC合計額の上限
- 最小HTLC金額
- commitment transactionの手数料
- channel reserve
- dust exposure
- CLTV expiry

つまり、HTLCは無限に置けるものではありません。ここが攻撃面になります。

## 前提：HTLCは正当に保留されることもある

HTLCが未解決のまま残ること自体は、必ずしも悪ではありません。

たとえばhodl invoiceでは、受取側が支払いを受け入れたあと、preimageをすぐに出さず、外部条件を確認してからsettleまたはcancelします。

```text
支払いを受け入れる
  ↓
preimageをまだ出さない
  ↓
HTLCが保留される
  ↓
条件を満たせばsettle
  ↓
満たさなければcancel
```

この仕組みは在庫確認、外部承認、atomic swapなどに使えます。

Channel Jammingが厄介なのは、正当な保留と攻撃的な保留が、経路上の中継ノードからは似て見えることです。

## Channel Jamming Attackとは何か

Channel Jamming Attackは、未解決HTLCを意図的に残し、経路上のチャネル資源を占有する攻撃です。

典型的には、攻撃者が送金側と受取側を用意します。

```text
攻撃者S
  |
  v
中継ノードA
  |
  v
標的チャネル
  |
  v
中継ノードB
  |
  v
攻撃者R
```

攻撃者Sは、攻撃者Rへ向けて支払いを送ります。しかしRはpreimageを出しません。すると、経路上のHTLCが未解決のまま残ります。

この間、経路上のチャネルでは、流動性やHTLC枠が使われ続けます。

## Liquidity jamming

Liquidity jammingは、チャネルの流動性を拘束する攻撃です。

攻撃者は比較的大きめの支払いを経路に流し、受取側でpreimageを出さずに保留します。

```text
1 BTCのHTLCを長い経路に流す
  ↓
各ホップのチャネルで1 BTC相当の流動性が拘束される
  ↓
他の支払いが通りにくくなる
```

攻撃者の資金は1 BTCでも、20ホップの経路を使えば、経路上では合計20 BTC分の流動性に影響を与えられます。

もちろん、攻撃者自身の資金も拘束されます。そのため完全に無料ではありません。ただし、支払いを成功させなければ通常の中継手数料を払わずに済みやすい点が問題です。

## HTLC jamming

HTLC jammingは、金額ではなくHTLC枠を狙う攻撃です。

BOLT #2では、通常のチャネルで受け入れ可能な未解決HTLC数に上限があります。`max_accepted_htlcs`は最大483です。

攻撃者は小額HTLCを大量に流し、受取側で保留します。

```text
小額HTLC 1
小額HTLC 2
小額HTLC 3
...
小額HTLC 483

→ HTLC枠が埋まる
```

この攻撃では、大きな資金量よりも「未解決HTLCの数」が重要になります。

小額でもHTLC枠は1つ使います。そのため、金額面では安く見える攻撃でも、チャネルの支払い処理能力を大きく落とせます。

## 攻撃の具体的な流れ

HTLC jammingを例にすると、攻撃は次のように進みます。

```text
攻撃者Sが攻撃者R宛てに小額支払いを作る
  ↓
標的チャネルを通る経路を選ぶ
  ↓
経路上にHTLCが作られる
  ↓
攻撃者Rはpreimageを出さない
  ↓
HTLCが未解決のまま残る
  ↓
同じことを繰り返す
  ↓
標的チャネルのHTLC枠が埋まる
  ↓
他の支払いが失敗しやすくなる
```

Liquidity jammingの場合は、同じ流れで大きめの金額を使い、HTLC枠ではなく流動性を拘束します。

## なぜ攻撃コストが低くなりやすいのか

Channel Jammingで問題になるのは、攻撃者が経路上の資源を使わせる一方で、支払いを成功させなければ中継手数料を払わずに済みやすいことです。

通常のルーティング手数料は、成功した支払いに対して発生します。

```text
支払い成功
  ↓
中継ノードが手数料を得る
```

しかしjammingでは、攻撃者は支払いを成功させません。

```text
HTLCを作る
  ↓
長く保留する
  ↓
最後に失敗させる
  ↓
中継ノードはリソースを使ったが、手数料を得にくい
```

攻撃者の資金は保留中に拘束されます。したがって完全に無料ではありません。

それでも、攻撃対象に与える影響に比べて攻撃コストが低くなりやすい点が問題です。

## なぜ中継ノードはすぐにfailできないのか

「怪しいHTLCなら中継ノードがすぐfailすればいい」と思うかもしれません。

しかし、中継ノードはそんなに自由には動けません。

```text
A -- incoming HTLC --> B -- outgoing HTLC --> C
```

Bは、Cへ出したoutgoing HTLCがまだ成功する可能性がある間、Aから受け取ったincoming HTLCを先にfailすると危険です。

危険な順序はこうです。

```text
1. BがAへ先にfailを返す
2. A→B のHTLCは失敗扱いになる
3. その後、Cがpreimageを出す
4. B→C のHTLCが成功する
5. BはCへ支払う
6. しかしAからは回収できない
```

結果として、Bだけが損します。

```text
Aから回収できない
Cには支払う
→ Bが損する
```

そのためBOLT #2では、forwarding HTLCについて、安全な順序が要求されます。中継ノードは、対応するoutgoing HTLCの削除が不可逆にコミットされるまで、incoming HTLCをfailしてはいけません。

これがChannel Jammingの土台です。下流側が保留している間、上流側も簡単には片付けられません。

## hodl invoiceとの違い

hodl invoiceとChannel Jammingは、どちらも「preimageをすぐに出さない」という点では似ています。

しかし目的が違います。

| 観点 | hodl invoice | Channel Jamming |
| --- | --- | --- |
| 目的 | 条件成立まで支払いを保留する | 経路上の資源を占有する |
| 量 | 必要な範囲 | 大量 |
| 時間 | アプリケーション上必要な短時間 | 長く引き延ばす |
| 対象 | 自分宛ての支払い | 他人のチャネルや経路 |
| 結果 | settleまたはcancelで解放 | 他の支払いを妨害 |

hodl invoiceは正当な機能です。Channel Jammingは、その保留という性質を攻撃目的に使います。

## 対策が難しい理由

Channel Jammingへの対策は簡単ではありません。

### 失敗支払いにも課金するべきか

攻撃者は失敗支払いを使うため、失敗した支払いにも何らかのコストを課す案があります。

たとえば、upfront feeやhold feeのような考え方です。

ただし、Lightningでは正当な支払いも失敗します。流動性不足、経路情報の古さ、一時的なノード停止などが原因です。失敗に強く課金すると、普通のユーザー体験が悪くなる可能性があります。

### 中継ノードは支払いの全体像を知らない

Lightningはonion routingを使います。中継ノードは、経路全体や支払いの目的を知りません。

これはプライバシー上重要ですが、攻撃判定を難しくします。

```text
このHTLCは正当なhodl invoiceなのか
攻撃者が保留しているのか
一時的に遅いだけなのか
```

中継ノードからは、これらを完全には区別できません。

### 厳しくしすぎると正当な用途を壊す

HTLCの保留を短く制限すれば、jammingは難しくなります。

しかし、hodl invoiceやatomic swapのような正当な用途も使いにくくなります。

対策は、セキュリティ、手数料、プライバシー、UXのバランス問題になります。

## 議論されている対策の方向性

ここでは概要だけ触れます。

### upfront fee

支払いが成功するかどうかに関係なく、経路を試す時点で小さな手数料を払う考え方です。

攻撃者にコストを課せますが、正当な失敗支払いにもコストが発生します。

### hold fee

HTLCを保留している時間に応じて課金する考え方です。

リソース占有に対してコストを対応させやすい一方、実装やプロトコル上の設計は簡単ではありません。

### reputation

支払いの失敗や保留の傾向をもとに、ピアや経路の信頼度を調整する考え方です。

ただし、プライバシーやSybil攻撃とのトレードオフがあります。

### local policy

各ノードが、保留時間、HTLC数、金額、ピアごとの制限などをローカルポリシーとして調整する方法です。

即効性はありますが、ネットワーク全体の根本解決にはなりにくいです。

## よくある誤解

### Channel Jammingはノードを落とす攻撃である

正確には、ノード停止というより支払い経路のリソースを占有する攻撃です。

### 大金がないと攻撃できない

Liquidity jammingでは資金量が重要になりますが、HTLC jammingでは小額HTLCを大量に使うことで成立し得ます。

### 怪しいHTLCはすぐキャンセルすればよい

中継ノードは、下流側HTLCが安全に片付く前に上流側をfailすると、自分だけ損する可能性があります。

### hodl invoiceは攻撃機能である

違います。hodl invoiceは正当な保留機能です。ただし、HTLCを保留するという性質はjammingの理解に役立ちます。

## まとめ

Channel Jamming Attackは、Lightningの支払いが有限なチャネル資源を使うことを悪用します。

- HTLCは無制限に置けるものではありません
- 攻撃者は未解決HTLCを意図的に残します
- liquidity jammingは流動性を拘束します
- HTLC jammingはHTLC枠を埋めます
- 中継ノードは安全上、HTLCを雑にfailできません
- 対策は、失敗支払いへの課金、プライバシー、正当な保留との区別が絡むため難しいです

Channel Jammingを理解すると、Lightningが単に「速い支払いネットワーク」ではなく、流動性と時間を扱うプロトコルであることが見えてきます。

## 参考資料

- [Bitcoin Optech: Channel jamming attacks](https://bitcoinops.org/en/topics/channel-jamming-attacks/)（2026-06-29確認）
- [BOLT #2: Peer Protocol](https://github.com/lightning/bolts/blob/master/02-peer-protocol.md)（2026-06-29確認）
- [BOLT #4: Onion Routing Protocol](https://github.com/lightning/bolts/blob/master/04-onion-routing.md)（2026-06-29確認）
- [LND API: AddHoldInvoice](https://api.lightning.community/api/lnd/invoices/add-hold-invoice/index.html)（2026-06-29確認）

## 更新履歴

- 2026-06-29: 初版
