---
title: "JIT Channelとは何か — チャネルなしで最初のLightning支払いを受け取る仕組み"
emoji: "⚡"
type: "tech"
topics: ["bitcoin", "lightning", "lsp"]
published: true
---

## はじめに

Lightning Networkで支払いを受け取るには、自分へ向いた流動性、つまりinbound liquidityが必要です。

しかし、Lightningを使い始めたばかりのウォレットにはチャネルがありません。チャネルを開くにはオンチェーンのBitcoinとマイニング手数料が必要です。

```text
Lightningで受け取りたい
  ↓
受け取るにはチャネルとinbound liquidityが必要
  ↓
チャネルを用意するにはBitcoinと開設手数料が必要
  ↓
まだBitcoinを持っていない
```

**JIT Channel（Just-In-Time Channel）**は、この最初の受取で生じる問題を解決する仕組みです。

送金者からLightning支払いが到着したタイミングで、LSP（Lightning Service Provider）が受取人とのチャネルを開きます。チャネル開設料は最初の受取金額から差し引かれるため、受取人は事前にチャネルやBitcoinを用意しなくてもLightningを使い始められます。

この記事では、まず「誰がどの資金を動かすのか」という全体像を整理します。そのあと、LSPS2で定義されている`jit_channel_scid`、HTLCの保留、チャネル開設、開設料の控除、信頼モデルを見ていきます。

## 先に結論

JIT Channelの要点は、次のとおりです。

- 送金者は、すでにLightning上で送金できる資金を持っています
- LSPは、公開Lightning Networkから支払いを受け取れる流動性を持っています
- LSPは、受取人とのチャネルを開くためのオンチェーン資金も持っています
- 受取人は、事前のLightning資金やオンチェーン資金を持っていなくても構いません
- LSPはincoming HTLCを一時的に保留し、その間に受取人とのチャネルを開きます
- LSPは新しいチャネルへ支払いを転送し、初回支払いからチャネル開設料を差し引きます
- チャネル開設後、受取人は受け取った残高を通常のLightning支払いに使えます
- funding transactionとpreimageのどちらを先に出すかによって、LSPと受取人の信頼関係が変わります

一言でまとめると、JIT Channelは、**最初の入金を受け取るためのチャネルを、その入金が到着したときに作る仕組み**です。

## JIT Channelの全体像

登場人物は次の3者です。

- **送金者（payer）**：受取人へLightning支払いを送る人
- **LSP**：チャネルと流動性を提供する事業者またはノード
- **受取人（client）**：JIT Channelを使って最初の支払いを受け取るウォレット

単純化した流れは、次のようになります。

```text
送金者
  │ Lightning支払い
  ▼
LSP
  │ 自分のオンチェーン資金でチャネルを開設
  │ 開設したチャネルへ支払いを転送
  ▼
受取人
```

ここでLSPは、単に送金者と受取人を紹介する仲介者ではありません。自分の資金を使って受取人へチャネルを提供する、流動性プロバイダーです。

LSPには次のものが必要です。

- 公開Lightning Networkから支払いを受け取るためのチャネルと流動性
- 新しいチャネルのfunding transactionに使うオンチェーンUTXO
- funding transactionのマイニング手数料
- incoming HTLCを保留し、チャネル開設後に転送する機能

LSPが送金者と直接チャネルを持っている必要はありません。送金者から既存のLightning経路を通してLSPまで支払いを届けられればよいからです。

## LSPは何を受け取り、何を提供するのか

送金者が受取人へ100,000 satを支払い、LSPのチャネル開設料が1,000 satだったとします。

LSPは既存のLightningチャネルを通して、送金者側から100,000 satを受け取ります。同時に、自分のオンチェーン資金で受取人との新しいチャネルを開き、そのチャネル上で99,000 satを受取人側へ移します。

```text
既存のLightning経路

送金者側 ── 100,000 sat ──> LSP


新しく開いたチャネル

LSP ── 99,000 sat ──> 受取人


差額

100,000 sat - 99,000 sat = 1,000 sat
                              開設料
```

LSPの資金の動きを整理すると、次のようになります。

- 既存チャネル上では、送金者側からLSP側へ残高が移ります
- 新しいチャネル上では、LSP側から受取人側へ残高が移ります
- LSPは差額としてチャネル開設料を受け取ります

つまり、LSPが受取額を無償で提供しているわけではありません。

LSPは、自分が新しいチャネルへ投入したオンチェーン資金を受取人側のLightning残高へ変えます。その代わり、送金者から既存チャネル上のLightning残高を受け取り、さらに開設料を得ます。

受取人だけを見ると、事前資金なしで99,000 satを受け取った状態になります。

## なぜ通常は最初の支払いを受け取れないのか

Lightningチャネルには、固定されたcapacityと、その中で両者の間を移動する残高があります。

たとえば、LSPが100,000 satを全額出して受取人とのチャネルを開いた直後は、概念的に次の状態です。

```text
LSP [100,000 sat] ───────── [0 sat] 受取人
```

この状態なら、LSPから受取人へ最大100,000 sat近くを送れます。受取人から見ると、LSP側にある残高がinbound liquidityです。

支払いが転送されると、チャネル内の残高が受取人側へ移ります。

```text
LSP [1,000 sat] ─────── [99,000 sat] 受取人
```

この99,000 satは、受取人が次のLightning支払いに使えるoutbound liquidityになります。

一方、残高が受取人側へ大きく移ったため、同じチャネルでさらに受け取れる余地は小さくなります。LSPがチャネルを大きめに作らない限り、JIT Channelは主に「最初の受取」を可能にする仕組みです。最初の受取後も無制限に受け取れるわけではありません。

## まだ存在しないチャネルをどう指定するのか

ここまでの説明には、一見すると不思議な点があります。

支払いがLSPへ到着する時点では、LSPと受取人の間にまだチャネルがありません。では、送金者は存在しないチャネルを、どうやって支払い経路へ含めるのでしょうか。

ここで使われるのが、**`jit_channel_scid`**です。

`jit_channel_scid`は実在するチャネルを指すSCIDではなく、LSPが発行するJIT Channel予約用の識別子です。

受取人は、この識別子とLSPのnode IDをBOLT 11 invoiceのroute hintへ入れます。

```text
送金者
  │ 公開Lightning Network上の経路
  ▼
LSP
  │ next hop: jit_channel_scid
  ▼
受取人
```

送金者は、公開ネットワーク上の経路を使ってLSPまでHTLCを届けます。LSPはonion payloadに含まれる次ホップのSCIDを確認し、それが自分の発行した`jit_channel_scid`なら、通常のチャネル転送ではなくJIT Channelの処理を開始します。

つまり、送金者が存在しないチャネルへ直接送っているのではありません。LSPまで支払いを届け、その先の処理をLSPが引き受けます。

## LSPS2による処理の流れ

JIT Channelの交渉は、bLIP-52として公開されている**LSPS2: JIT Channel Negotiation**で定義されています。

大まかな処理は次のとおりです。

```text
受取人がLSPの料金条件を取得
  ↓
JIT Channelを予約
  ↓
LSPがjit_channel_scidを発行
  ↓
受取人がroute hint付きinvoiceを作成
  ↓
送金者が支払う
  ↓
LSPがincoming HTLCを保留
  ↓
LSPが受取人とのチャネルを開く
  ↓
開設料を差し引いてHTLCを転送
  ↓
受取人がpreimageを返す
  ↓
支払い完了
```

### 1. 料金条件を取得する

受取人は、LSPとのBOLT 8接続上で`lsps2.get_info`を呼びます。

LSPは、次のような条件を返します。

- 最低開設料
- 受取金額に対する比例料金
- 料金条件の有効期限
- 受け付ける最小・最大支払金額
- チャネルの最低存続期間
- 許容する`to_self_delay`の上限
- LSPがその条件を提示したことを検証する`promise`

開設料は、概念的には次のように計算します。

```text
比例料金 = ceil(支払金額 × proportional / 1,000,000)

opening_fee = max(最低開設料, 比例料金)
```

この開設料は、funding transactionのマイニング手数料と必ず同額になるわけではありません。LSPが提示するサービス料金であり、オンチェーン費用、資金拘束、チャネル提供のリスクなどを含む料金設計にできます。

### 2. JIT Channelを予約する

受取人は、選んだ料金条件を`lsps2.buy`でLSPへ返します。固定額invoiceを作る場合は、受取予定額も指定できます。

LSPは、成功すると主に次の情報を返します。

```json
{
  "jit_channel_scid": "29451x4815x1",
  "lsp_cltv_expiry_delta": 144,
  "client_trusts_lsp": false
}
```

- `jit_channel_scid`：JIT Channel予約を識別するSCID
- `lsp_cltv_expiry_delta`：LSPから受取人までに必要なCLTV差
- `client_trusts_lsp`：受取人がLSPを信頼する処理順序を要求するか

この時点では、まだ実際のチャネルは作られていません。

### 3. Invoiceを作る

受取人は、LSPのnode IDと`jit_channel_scid`を1ホップのroute hintとしてBOLT 11 invoiceへ入れます。

初回支払いのroute hintでは、通常のbase feeとproportional feeを0にします。LSPの料金は、通常のルーティング手数料ではなく、事前合意した`opening_fee`として受取額から差し引くためです。

また、通常より大きなCLTVの余裕を持たせます。LSPへHTLCが到着してから、チャネル開設やfunding transactionの確認待ちを行う時間が必要だからです。

固定額invoiceではMPPを使用できます。金額なしinvoiceでは、LSPS2はMPPを無効にします。MPPの合計金額は受取人だけが読める情報に依存するため、中継地点にいるLSPは、すべてのpartが揃ったか判断できないからです。

### 4. LSPがHTLCを保留する

送金者がinvoiceを支払うと、HTLCは公開Lightning Networkを通ってLSPへ到着します。

LSPは、次ホップとして指定された`jit_channel_scid`を見て、対応するJIT Channel予約を特定します。

```text
送金者からのincoming HTLC
             ↓
            LSP
             ↓
 next hopがjit_channel_scid
             ↓
    HTLCを一時的に保留
```

この時点では、まだ受取人へ転送できません。LSPはincoming HTLCを失敗させずに保留し、その間に受取人とのチャネルを開きます。

MPPの場合は、同じpayment hashのpartを集め、合計が予定額に達するまで待ちます。partが揃わない場合や有効期限を過ぎた場合は、支払いを失敗させます。

### 5. 受取人とのチャネルを開く

LSPは、標準的なBOLT #2のチャネル開設フローを使って、受取人とのチャネルを開きます。

LSPS2では、次の条件を使用します。

- private channelとして開く
- `option_scid_alias`を使用する
- 必要に応じて0-confで利用を開始する
- 受取人が提示する`to_self_delay`は、合意した上限以下にする

チャネル容量はLSPが決めます。ただし、少なくとも開設料控除後の支払いと、LSP側に必要なchannel reserveを収容できる必要があります。

`option_scid_alias`を使うことで、funding transactionがまだ承認されていないチャネルもSCID aliasで参照できます。

### 6. 開設料を差し引いて転送する

チャネルが利用可能になると、LSPは保留していたHTLCを受取人へ転送します。

通常のLightning転送と異なり、JIT ChannelではLSPが開設料を差し引くため、受取人へ届く金額はonion内の指定額より小さくなります。

```text
送金者からLSP: 100,000 sat
開設料:           1,000 sat
LSPから受取人:  99,000 sat
```

この控除には、bLIP-25で定義された`extra_fee`が使われます。

受取人は、実際のHTLC金額と`extra_fee`を確認し、差し引かれた合計が事前に合意した`opening_fee`以下であることを検証します。LSPが合意していない金額を黙って差し引く仕組みではありません。

受取人がHTLCを受け入れてpreimageを返すと、preimageが経路を逆向きに伝わり、初回支払いが完了します。

## funding transactionとpreimageの信頼モデル

JIT Channelで最も難しいのは、次の2つをどの順番で渡すかです。

```text
LSPが提供するもの:
有効なfunding transactionとチャネル資金

受取人が提供するもの:
支払いを確定するpreimage
```

LSPS2では、2つの信頼モデルを定義しています。

### LSPが受取人を信頼するモデル

仕様が推奨するデフォルトです。

```text
LSPがfunding transactionをブロードキャスト
  ↓
受取人がfunding outputを確認
  ↓
LSPがHTLCを転送
  ↓
受取人がpreimageを返す
```

受取人は、funding transactionがmempoolへ入るまで待つことも、1承認以上待つこともできます。そのため、受取人は自分のリスク許容度に応じてLSPへの信頼を小さくできます。

一方、LSPにはリスクがあります。悪意ある受取人がチャネルを開かせたあとpreimageを返さなければ、LSPは開設料を受け取れず、自分の資金だけをチャネルへロックされる可能性があります。

### 受取人がLSPを信頼するモデル

LSPが`client_trusts_lsp`を指定した場合のモデルです。

```text
LSPがチャネル開設情報を提示
  ↓
LSPがHTLCを転送
  ↓
受取人がpreimageを返す
  ↓
LSPがfunding transactionをブロードキャスト
```

こちらは、LSPがpreimageを確認してからfunding transactionを公開できるため、LSPのリスクを小さくできます。

しかし、受取人はLSPを信頼する必要があります。LSPが無効なfunding transactionを提示したり、実際にはブロードキャストしなかったりすると、受取人は支払いのpreimageを渡したのに、有効なチャネル資金を得られない可能性があります。

### どちらも相手を信頼しない場合

```text
LSP:
preimageを見てからfunding transactionを出したい

受取人:
funding transactionを確認してからpreimageを出したい
```

この場合はデッドロックします。

LSPS2は、funding transactionとpreimageを完全に原子的に交換する仕組みではありません。どちらが先にリスクを取るかを明示し、受取人が信頼モデルを確認できるようにしています。

## チャネル開設後はどうなるのか

初回支払いが完了すると、JIT Channelは通常のprivate channelとして利用できます。

受取人は、チャネル開設時に割り当てられたSCID aliasを、その後のinvoiceのroute hintに使います。最初に使った`jit_channel_scid`と、開設後のSCID aliasが同じとは限りません。

初回支払いによって受取人側へ移った残高は、受取人のoutbound liquidityになります。

```text
初回受取前

LSP [チャネル資金] ───── [0] 受取人


初回受取後

LSP [残り] ───── [受取額 - 開設料] 受取人
```

受取人がその後LSP側へ支払えば、残高がLSP側へ戻り、再びinbound liquidityが増えます。

## 攻撃と失敗ケース

### Prepayment probingでチャネルが開く

送金者は、実際の支払い前に経路や手数料を確認するため、偽のpayment hashを使ったprobeを送ることがあります。

LSPがprobeを本物のJIT支払いと区別できなければ、支払いが成功しないのにチャネルを開いてしまう可能性があります。LSPが受取人を信頼するモデルでは、LSPは開設料を受け取れず、資金だけをロックされるおそれがあります。

LSPS2はこの問題を明記していますが、具体的な評価や防御方法はLSP実装に委ねています。

### 受取人がオフラインになる

JIT Channelを開くには、受取人がLSPへ接続してチャネル開設メッセージを処理する必要があります。

受取人が`funding_signed`を返す前に切断した場合、LSPはincoming HTLCを一時的なチャネル障害として失敗させます。送金者は、受取人がオンラインへ戻ったあと再試行できます。

### 料金条件が期限切れになる

LSPが提示する料金条件には`valid_until`があります。

受取人は、この期限を超えないinvoiceを作る必要があります。期限後にHTLCが到着した場合、LSPは古い料金条件でチャネルを開く義務を負いません。

### MPPのpartが揃わない

固定額invoiceでMPPを使う場合、LSPは同じpayment hashのpartを保留します。

必要な合計額へ達しなければチャネルを開けません。一定時間内にpartが揃わない場合、LSPは支払いを失敗させます。保留中はLSPと経路上のHTLC枠や流動性を使用します。

### 0-confチャネルを使う

0-confなら、オンチェーン承認を待たずに初回支払いを転送できます。

その一方で、funding transactionはまだ承認されていません。受取人はLSPを信頼してすぐpreimageを返すか、mempoolやブロック承認を確認するかを選びます。確認を待つほどオンチェーン上の安全性は高まりますが、支払い完了までHTLCを長く保留することになります。

## よくある誤解

### JIT ChannelがBitcoinをLN上に新しく作る

JIT ChannelはBitcoinを作りません。

送金者側には、すでにLightning上で送れる資金が必要です。またLSPには、受取人とのチャネルを開くためのオンチェーン資金が必要です。

事前資金を持たなくてもよいのは受取人です。

### LSPは送金者と直接つながっている必要がある

LSPが送金者と直接チャネルを持つ必要はありません。

送金者から公開Lightning Network上の経路を使って、LSPまで支払いを届けられれば十分です。

### LSPはチャネル開設を紹介するだけである

LSPは紹介者ではなく、実際に自分のオンチェーンUTXOをfunding transactionへ入れ、チャネル資金と流動性を提供します。

### 受取人は初回受取後も同じだけ受け取れる

初回支払いによってチャネル残高が受取人側へ移るため、その分のinbound liquidityは消費されます。

チャネルが支払額より大きく作られている場合や、受取人が残高を使ってLSP側へ戻した場合を除き、続けて同額を受け取れるとは限りません。

### JIT Channelは必ずカストディである

チャネル開設後に受取人が自分の鍵とcommitment transactionを管理する構成なら、通常の自己管理型Lightningチャネルとして利用できます。

ただし、開設時にはLSPへの接続とサービス提供が必要であり、選択した信頼モデルによって一時的な信用リスクがあります。開設後の資金管理と、開設時のLSP依存は分けて考える必要があります。

## LSPS1との違い

LSPS1もLSPからチャネルを購入する仕様ですが、購入のタイミングと支払い方法が異なります。

| 項目 | LSPS1 | LSPS2 JIT Channel |
| --- | --- | --- |
| チャネル開設 | 事前に注文する | 初回支払いの到着時に行う |
| 開設料 | on-chainまたは既存のLightning資金で支払う | 初回受取額から差し引く |
| 受取人の事前資金 | 必要になる場合がある | なくても開始できる |
| 主な用途 | 計画的なチャネル購入 | 新規ウォレットの最初の受取 |
| Invoice | 通常のチャネル情報を使う | `jit_channel_scid`をroute hintに使う |
| 初回HTLC | 通常どおり転送する | LSPが保留し、チャネル開設後に転送する |

## 仕様と実装状況

2026年7月27日時点で、LSPS2はLightning全体の必須仕様であるBOLTではなく、bLIP-52として`Active`になっています。LSPとウォレットの相互運用を目的とした仕様です。一方、初回支払いから開設料を差し引く`extra_fee`を定義するbLIP-25のstatusは`Draft`です。

Lightning Dev Kitの`lightning-liquidity`には、LSPS2のclient側とservice側のロジックが実装されています。同ライブラリのドキュメントでは、service側サポートはbetaとされています。

実際の対応状況、料金、信頼モデル、0-confの扱いは、ウォレットやLSPによって異なります。

## まとめ

JIT Channelは、Lightningを初めて使う受取人のために、支払い到着時にチャネルを開く仕組みです。

- 送金者はLightning上の資金をLSPへ送ります
- LSPは公開ネットワークから支払いを受け取れる流動性を持ちます
- LSPは自分のオンチェーン資金で受取人とのチャネルを開きます
- `jit_channel_scid`によって、まだ存在しないチャネルへの予約をroute hintへ表現します
- LSPはincoming HTLCを保留し、その間にチャネルを開きます
- チャネル開設料は最初の受取金額から差し引かれます
- 受取人は事前のチャネルやBitcoinなしで最初の支払いを受け取れます
- 初回受取後の残高はoutbound liquidityになります
- funding transactionとpreimageの順番によって、LSPと受取人の信頼関係が変わります

JIT Channelの中心は、単に「自動でチャネルを開く」ことではありません。

送金者から既存のLightning経路で届いた資金をLSPが受け取り、自分のオンチェーン資金を新しいチャネルへ移し替えることで、事前資金を持たない受取人をLightningへ参加させる仕組みです。

## 参考資料

- [bLIP-52: LSPS2 JIT Channel Negotiation](https://github.com/lightning/blips/blob/master/blip-0052.md)（2026-07-27確認）
- [bLIP-50: LSPS0 Transport Layer](https://github.com/lightning/blips/blob/master/blip-0050.md)（2026-07-27確認）
- [bLIP-25: Forward less than onion value](https://github.com/lightning/blips/blob/master/blip-0025.md)（2026-07-27確認）
- [BOLT #2: Peer Protocol for Channel Management](https://github.com/lightning/bolts/blob/master/02-peer-protocol.md)（2026-07-27確認）
- [BOLT #11: Invoice Protocol for Lightning Payments](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md)（2026-07-27確認）
- [lightning-liquidity](https://docs.rs/lightning-liquidity)（2026-07-27確認）

## 更新履歴

- 2026-07-27: 初版
