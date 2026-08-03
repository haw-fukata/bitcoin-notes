---
title: "Lightningの支払いルートはどう決まるのか — InvoiceとGossipから経路を導出する"
emoji: "🗺️"
type: "tech"
topics: ["bitcoin", "lightning", "routing"]
published: true
---

## はじめに

Lightning Networkでは、送金者と受取人が直接チャネルを持っていなくても、複数のノードを経由して支払えます。

```text
送金者 Alice
  ↓
中継ノード Bob
  ↓
受取人 Carol
```

では、Aliceはどのチャネルを通ればCarolへ届くと知るのでしょうか。また、各ホップへ渡す金額とHTLCの期限は、どのように決めるのでしょうか。

この記事では、通常のLightning支払いについて、BOLT 11 invoiceを読み、gossipから作ったネットワークグラフを使って、支払いルートを導出するまでを整理します。

実際の支払いでは、導出したルートからSphinx形式のOnion Packetを作ります。ただし、Onion Packetの暗号化やRoute Blindingは別の記事で扱い、今回は「どのノードとチャネルを通り、各ホップへいくら・どの期限で渡すか」を決めるところまでに絞ります。

## 先に結論

通常のルート導出は、次のように進みます。

```text
BOLT 11 invoiceを読む
  ↓
受取人、金額、最終CLTVなどを取得
  ↓
gossipから作ったネットワークグラフを参照
  ↓
使えないチャネルを除外
  ↓
手数料、CLTV、成功確率などで候補を評価
  ↓
受取人側から逆向きに金額と期限を計算
  ↓
支払いルートを確定
```

重要な点は、次のとおりです。

- `channel_announcement`から公開チャネルの存在を知ります
- `channel_update`から方向ごとの手数料、CLTV差、HTLC範囲などを知ります
- invoiceから受取人、支払額、最終ホップのCLTV条件などを得ます
- 非公開チャネルがある場合はinvoiceのroute hintでグラフを補います
- 手数料とCLTVは受取人側から送金者側へ逆向きに積み上げます
- どの候補を選ぶかという探索・スコアリング方式は実装依存です
- 公開情報から実際の片方向残高は分からないため、導出したルートが成功する保証はありません

## ルート導出に必要な2種類の情報

送金者は、主に次の2つを組み合わせます。

1. 受取人が発行したBOLT 11 invoice
2. Lightningのgossipから作った公開ネットワークグラフ

```text
Invoice
  ├─ どこへ払うか
  ├─ いくら払うか
  └─ 最終ホップの条件

Gossip Graph
  ├─ どのノード間にチャネルがあるか
  ├─ 方向ごとの手数料
  ├─ 方向ごとのCLTV差
  └─ HTLCの最小・最大額
```

invoiceが目的地と支払い条件を示し、gossip graphがそこへ到達するための候補を示します。

## Invoiceから得る情報

BOLT 11 invoiceには、支払いルートを作るための情報が含まれます。

主に使うのは次の項目です。

- 受取人のnode ID
- 支払金額
- invoiceの有効期限
- `min_final_cltv_expiry_delta`
- 利用可能または必須のfeature
- 必要な場合はroute hint

### 受取人のnode ID

`n`フィールドがあれば、その公開鍵が受取人のnode IDです。`n`がなければ、送金者はinvoiceの署名から公開鍵を復元できます。

このnode IDが、通常の経路探索における目的地になります。

### 支払金額

金額指定invoiceなら、送金者はその金額が受取人へ届くルートを探します。

金額なしinvoiceの場合は、支払う側が金額を決めてから探索します。

### `min_final_cltv_expiry_delta`

これは、最後のHTLCについて受取人が要求する、現在のブロック高から期限までの最低差です。BOLT 11では`c`フィールドに入り、省略時は少なくとも18を使用します。

```text
現在のブロック高: 900,000
min_final_cltv_expiry_delta: 18

最終HTLCの最低期限: 900,018
```

### Route Hint

受取人が非公開チャネルだけで接続されている場合、そのチャネルは公開gossip graphに現れません。

そこでinvoiceの`r`フィールドに、公開ノードから受取人までの追加経路を入れます。

```text
公開グラフ                    非公開部分

Alice → Bob → Introduction ─────────→ Carol
                         route hint
```

送金者はroute hintをローカルのグラフへ一時的に追加し、公開部分とつなげて経路を探します。

## Gossipからネットワークグラフを作る

BOLT #7では、ノードとチャネルを発見するために主に3種類のgossipメッセージを使います。

| メッセージ | 主な役割 |
| --- | --- |
| `node_announcement` | node ID、接続先、featureなどを知らせる |
| `channel_announcement` | 2ノード間に公開チャネルが存在することを知らせる |
| `channel_update` | 方向ごとの転送条件を知らせる |

経路探索で特に重要なのは、`channel_announcement`と`channel_update`です。

### Channelは方向付きの2つのEdgeになる

Lightningチャネルは双方向ですが、転送条件は方向ごとに異なります。

```text
Alice ───────── Bob

グラフ上では:

Alice → Bob
Bob   → Alice
```

AliceからBobへ送る条件と、BobからAliceへ送る条件は同じとは限りません。それぞれのノードが、自分から相手へ転送するときの`channel_update`を送ります。

`channel_update`には、主に次の情報があります。

- その方向が現在無効か
- `fee_base_msat`
- `fee_proportional_millionths`
- `cltv_expiry_delta`
- `htlc_minimum_msat`
- `htlc_maximum_msat`

送金ノードはこれらをdirectional edgeとして保存し、ローカルのネットワークグラフを作ります。

## 候補から使えないEdgeを除外する

すべての公開チャネルを経路候補にできるわけではありません。

送金ノードは、少なくとも次の条件を確認します。

- `disable`フラグが設定されていないか
- 転送額が`htlc_minimum_msat`以上か
- 転送額が`htlc_maximum_msat`以下か
- 公開されているチャネル容量を超えていないか
- 必要なfeatureに対応しているか
- 手数料が送金者の上限内か
- CLTVの合計が送金者の許容範囲内か
- 自分が過去の失敗から利用しにくいと判断していないか

ただし、公開チャネルの実際の片方向残高はgossipされません。

```text
公開情報:
チャネル容量 1,000,000 sat

実際の残高:
Alice [950,000 sat] ───── [50,000 sat] Bob
```

総容量が1,000,000 satでも、BobからAliceへ送れるのは50,000 sat程度です。送金者はこの内部残高を正確には知りません。

したがって、グラフから分かるのは「通れる可能性があるルート」であり、成功が保証されたルートではありません。

## どのルートを選ぶかは実装依存

BOLTは、手数料やCLTVなどルートが満たすべき条件を定義します。一方、「候補の中からどれを選ぶか」というアルゴリズムを1つに固定していません。

実装は、たとえば次の要素を組み合わせて候補を評価します。

- 支払うルーティング手数料
- 合計CLTVによる資金拘束リスク
- 過去の成功・失敗から推定した成功確率
- 自分のローカルチャネル残高
- 経路長
- 特定ノードやチャネルへの集中回避

概念的には、次のようなコストを最小化します。

```text
経路コスト
  = ルーティング手数料
  + CLTVによるリスク
  + 失敗しやすさへのペナルティ
```

そのため、最も手数料が安いルートが常に選ばれるとは限りません。少し高くても、成功しやすいと推定されたルートが選ばれることがあります。

## 金額とCLTVは後ろから計算する

通るノード列が決まったら、各ホップのHTLC金額と期限を計算します。

この計算は受取人側から送金者側へ逆向きに行います。

理由は、中継ノードの手数料が「次のホップへ転送する金額」をもとに決まり、incoming HTLCの期限もoutgoing HTLCの期限へ`cltv_expiry_delta`を足して決まるからです。

### 手数料

中継手数料は次の式です。

```text
fee
  = fee_base_msat
  + amount_to_forward × fee_proportional_millionths / 1,000,000
```

整数除算では端数が切り捨てられます。

### 3ノードの計算例

AliceがBobを経由してCarolへ100,000 msatを送るとします。

```text
Alice → Bob → Carol
```

Carolのinvoiceは次の条件です。

```text
支払金額: 100,000 msat
min_final_cltv_expiry_delta: 18
```

Bobが`Bob → Carol`方向へ設定した条件を次のようにします。

```text
fee_base_msat: 1,000 msat
fee_proportional_millionths: 200 ppm
cltv_expiry_delta: 40
```

現在のブロック高は900,000とします。

#### Carolへ届くHTLC

Carolは100,000 msatを受け取る必要があります。

```text
Bob → Carol

amount_msat: 100,000
cltv_expiry: 900,000 + 18
             = 900,018
```

#### Bobが受け取るHTLC

BobはCarolへ100,000 msatを転送し、自分の手数料を受け取ります。

```text
比例手数料
  = 100,000 × 200 / 1,000,000
  = 20 msat

Bobの手数料
  = 1,000 + 20
  = 1,020 msat
```

したがって、AliceからBobへのHTLCは次のようになります。

```text
Alice → Bob

amount_msat: 100,000 + 1,020
             = 101,020

cltv_expiry: 900,018 + 40
             = 900,058
```

全体を並べると次のようになります。

| HTLC | 金額 | CLTV expiry |
| --- | ---: | ---: |
| Alice → Bob | 101,020 msat | 900,058 |
| Bob → Carol | 100,000 msat | 900,018 |

BobはCarolへ100,000 msatを払い、Aliceから101,020 msatを受け取るため、差額の1,020 msatが手数料になります。

またBobのincoming HTLCは、outgoing HTLCより40ブロック遅く期限切れになります。この差により、Carol側でオンチェーン解決が必要になっても、BobがAlice側から回収する時間を確保できます。

ホップが増える場合も、この計算を受取人側から順番に繰り返します。

## 導出されるルート情報

最終的に送金ノードは、概念的に次の情報を持ちます。

```text
Route
  ├─ Hop 1
  │    ├─ 次のnode ID
  │    ├─ 使用するchannel
  │    ├─ amount_to_forward
  │    └─ outgoing_cltv_value
  ├─ Hop 2
  │    └─ ...
  └─ Final Hop
       ├─ 受取金額
       └─ final CLTV
```

このルート情報をもとに、送金者は各ホップだけが自分向けの転送情報を読めるOnion Packetを構築します。

送金者はルート全体を知っていますが、中継ノードが知るのは基本的に前後のホップと、自分が転送するための情報です。

## 支払いに失敗したら再探索する

導出したルートに十分な片方向残高がなければ、支払いは失敗します。ノードがオフラインだったり、gossipが古かったりする場合もあります。

送金ノードは失敗情報を受け取ると、該当するチャネルやノードへの評価を更新し、別の候補を探します。

```text
ルート1を試す
  ↓ 失敗
失敗情報をローカル評価へ反映
  ↓
ルート2を導出
  ↓
再試行
```

この成功・失敗の履歴は、通常は各送金ノードがローカルに持つ情報です。同じinvoiceを払っても、送金ノードごとに異なるルートが選ばれ得ます。

## MPPでは複数ルートを作る

invoiceがBasic MPPに対応している場合、送金額を複数のpartへ分割し、別々のルートで送れます。

```text
             ┌→ Route A: 60,000 msat ─┐
Alice ───────┤                         ├→ Carol
             └→ Route B: 40,000 msat ─┘
```

各partについて、ここまで説明した制約確認と逆向き計算を行います。MPPの受取側でのまとめ方や`payment_secret`の役割は、別のテーマとして扱えます。

## 通常ルートとRoute Blindingの違い

通常のroute hintでは、非公開部分のnode IDやchannel IDなどを送金者へ渡します。そのため送金者は、公開経路だけでなく受取人までの追加経路も知ります。

Route Blindingでは、受取人が経路の後半をblinded pathとして作り、送金者には実際のnode IDやchannel IDを直接見せません。

```text
通常ルート:
送金者が目的地までの実ノード列を知る

Route Blinding:
送金者が通常探索する部分
  ＋
受取人が事前に作ったblinded path
```

Route Blindingの記事では、introduction node、blinded node ID、暗号化された転送情報を使い、送金者が経路後半の実体を知らずに支払える理由を整理します。

## よくある誤解

### BOLTが最短経路探索アルゴリズムを決めている

BOLTはgossip情報、手数料、CLTVなどのルールを定義しますが、候補の探索・スコアリング方式は実装依存です。

### 公開チャネル容量が十分なら支払いは成功する

公開される容量はチャネル全体の容量です。実際の片方向残高は公開されないため、必要な方向の流動性が足りない可能性があります。

### 中継ノードが経路全体を決める

通常のsource routingでは、送金者がルート全体を導出します。中継ノードはOnion Packetに従って次のホップへ転送します。

### 最も手数料が安いルートが必ず選ばれる

実装はCLTVや推定成功確率なども評価できます。最安ルートより成功しやすい候補を選ぶ場合があります。

### Route Hintは公開gossipへ追加される

route hintはinvoiceを受け取った送金者が、その支払いのために使う追加情報です。通常の公開gossipとしてネットワーク全体へ配布されるものではありません。

## まとめ

通常のLightningルートは、invoiceの支払い条件と、gossipから作ったネットワークグラフを組み合わせて導出します。

- invoiceから受取人、金額、最終CLTVなどを取得します
- `channel_announcement`から公開チャネルの存在を知ります
- `channel_update`から方向ごとの転送条件を知ります
- 非公開部分はroute hintでグラフを補います
- 利用できないedgeを除外し、手数料や成功確率などで候補を評価します
- 金額とCLTVは受取人側から逆向きに計算します
- 実際の片方向残高は公開されないため、失敗時は別ルートを探します
- 経路探索とスコアリングの詳細は実装ごとに異なります

通常のルート導出では、送金者が受取人までの経路を組み立てます。次の記事では、経路の後半を受取人側で隠して作るRoute Blindingを扱います。

## 参考資料

- [BOLT #7: P2P Node and Channel Discovery](https://github.com/lightning/bolts/blob/master/07-routing-gossip.md)（2026-08-03確認）
- [BOLT #11: Invoice Protocol for Lightning Payments](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md)（2026-08-03確認）
- [BOLT #4: Onion Routing Protocol](https://github.com/lightning/bolts/blob/master/04-onion-routing.md)（2026-08-03確認）
- [BOLT #2: Peer Protocol for Channel Management](https://github.com/lightning/bolts/blob/master/02-peer-protocol.md)（2026-08-03確認）

## 更新履歴

- 2026-08-03: 初版
