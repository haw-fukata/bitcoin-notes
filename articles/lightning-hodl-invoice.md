---
title: "hodl invoiceとは何か — Lightningで支払いを保留する仕組み"
emoji: "🧾"
type: "tech"
topics: ["bitcoin", "lightning", "security"]
published: true
---

## はじめに

Lightningの支払いは、受取人がpreimageを公開すると完了します。では、受取人がpreimageをすぐに公開しなかったらどうなるのでしょうか。

その挙動をアプリケーション機能として使うのが**hodl invoice**です。

hodl invoiceでは、支払いが到着してもすぐにはsettleしません。受取側はHTLCを一度受け入れ、外部条件が満たされたあとでsettleするか、条件を満たさなければcancelします。

この記事では、hodl invoiceを通して「HTLCを保留する」とはどういうことかを見ます。次の記事で扱うChannel Jamming Attackの前に、まずは正当な保留の仕組みを理解します。

この連載では、次の順番でLightningのHTLC保留とjammingを見ていきます。

1. [HTLCの制約](/articles/lightning-htlc-constraints)
2. hodl invoice
3. [Channel Jamming Attackとは何か](/articles/lightning-channel-jamming-attack)

## 先に結論

hodl invoiceの要点は、次のとおりです。

- 通常のinvoiceでは、支払いが到着すると受取側はpreimageを出してsettleします
- hodl invoiceでは、受取側がpreimageをすぐに出しません
- 支払いは`accepted`のような保留状態になります
- 受取側は後からpreimageを渡してsettleできます
- 条件を満たさなければcancelできます
- 経路上ではHTLCが未解決のまま残るため、チャネルの流動性やHTLC枠を一時的に使います
- 正当な機能ですが、大量・長時間・他人の経路を狙って使うとChannel Jammingに似た状態を作れます

hodl invoiceは攻撃機能ではありません。ただし、HTLCを保留する仕組みを理解するには、とても良い題材です。

## 通常のLightning invoiceの流れ

通常のLightning支払いでは、受取人がinvoiceを作ります。invoiceにはpayment hash、金額、期限、経路ヒントなどが含まれます。

送金者はinvoiceを読み、受取人へ向かう経路を探して支払います。

```text
受取人がinvoiceを作る
  ↓
送金者が支払う
  ↓
経路上にHTLCが作られる
  ↓
受取人がpreimageを公開
  ↓
HTLCが逆向きにsettleされる
  ↓
支払い完了
```

このとき、preimageを知っているのは受取人です。受取人がpreimageを出すことで、支払いが成功します。

## hodl invoiceとは何か

hodl invoiceは、支払いが到着しても受取側がすぐにpreimageを出さないinvoiceです。

「hodl」はBitcoin界隈でよく使われる俗語ですが、ここでは「hold invoice」とほぼ同じ意味で捉えると分かりやすいです。つまり、支払いを一時的に保留します。

流れはこうです。

```text
受取側がhodl invoiceを作る
  ↓
送金者が支払う
  ↓
受取側はHTLCを受け入れる
  ↓
preimageはまだ出さない
  ↓
支払いは保留状態になる
  ↓
あとでsettleまたはcancelする
```

LNDでは、`AddHoldInvoice`でhold invoiceを作り、`SettleInvoice`でpreimageを渡して決済を確定し、`CancelInvoice`で取り消せます。

## preimageを出さないとはどういうことか

Lightningの支払いは、preimageが公開されるまで最終的には成功しません。

受取側がpreimageを出さない間、経路上の各ノードではHTLCが未解決のまま残ります。

```text
送金者 A
  |
  | HTLC
  v
中継ノード B
  |
  | HTLC
  v
受取人 C

Cはpreimageをまだ出さない
→ A→B と B→C のHTLCが残る
```

この状態は、支払いが失敗したわけでも成功したわけでもありません。条件付きの支払いが保留されている状態です。

## accepted、settled、canceled

実装によって名前は少し異なりますが、hold invoiceでは次のような状態を意識すると分かりやすいです。

| 状態 | 意味 |
| --- | --- |
| open | invoiceは作られているが、まだ支払いが来ていない |
| accepted | HTLCは到着したが、preimageはまだ出していない |
| settled | preimageを出して支払いが完了した |
| canceled | 支払いを取り消した |

重要なのは`accepted`です。

通常のinvoiceでは、支払いが来たらすぐsettledへ進みます。hodl invoiceでは、acceptedの状態でアプリケーションの判断を待てます。

## hodl invoiceの正当な用途

hodl invoiceには正当な用途があります。

### 在庫確認

ECサイトで、支払いが可能かどうかを先に確認したい場合があります。

```text
支払いを受け付ける
  ↓
在庫を確認する
  ↓
在庫があればsettle
  ↓
なければcancel
```

先にsettleしてしまうと、在庫がなかった場合に返金処理が必要になります。hodl invoiceなら、支払いを確定する前に条件を確認できます。

### 外部システムの承認

別のAPI、配送システム、本人確認、予約システムなどの結果を待ってから支払いを確定したいケースがあります。

この場合も、支払いを保留し、外部条件が満たされればsettleします。

### atomic swap

atomic swapでは、別のチェーンや別の支払い条件と連動させるために、preimageの公開タイミングが重要になります。

hodl invoiceは、別の条件が満たされるまでLightning側の支払いを保留する部品として使えます。

### escrow的な処理

サービスが条件成立まで支払いを確定しない、という使い方もできます。

ただし、LightningのHTLCには期限があります。長期間のエスクローをそのままLightningのHTLCで実現するのは向いていません。短時間の条件確認と考えるほうが自然です。

## 経路上では何が保留されるのか

hodl invoiceで保留されるのは、受取人のアプリケーション状態だけではありません。経路上のHTLCも未解決のまま残ります。

```text
A → B → C

Cがhodl invoiceでpreimageを出さない
  ↓
B→C のoutgoing HTLCが残る
  ↓
A→B のincoming HTLCも残る
```

中継ノードBから見ると、B→CのHTLCがまだ成功する可能性があります。そのため、BはA→BのHTLCを雑にfailできません。

この性質が、Channel Jamming Attackを理解するうえで重要です。

## どのリソースを使うのか

hodl invoiceで支払いを保留すると、経路上では次のリソースを使います。

- HTLC枠
- チャネルの流動性
- commitment transaction上の状態領域
- CLTV期限までの時間
- ノードの管理・監視コスト

特に重要なのは、HTLC枠と流動性です。

小額支払いでも、未解決HTLCとして残る限り、HTLC数の枠を使います。大きな支払いなら、チャネル残高も拘束します。

## 長く保留しすぎると何が問題になるのか

hodl invoiceは便利ですが、長く保留すればするほど経路上のノードに負担をかけます。

問題は主に3つです。

### 1. 他の支払いが通りにくくなる

HTLC枠や流動性が使われている間、同じチャネルを通る他の支払いが失敗しやすくなります。

### 2. 中継ノードが手数料を得られない

Lightningの中継手数料は、基本的に支払いが成功したときに発生します。保留されたまま最後にcancelされると、中継ノードは長時間リソースを使われても手数料を得にくいです。

### 3. 期限管理が必要になる

HTLCにはCLTV expiryがあります。期限に近づくと、ノードは安全のためにオンチェーン解決を考える必要があります。

hodl invoiceを使うアプリケーションは、期限ぎりぎりまで放置してはいけません。

## hodl invoiceとChannel Jammingの似ている点

hodl invoiceとChannel Jamming Attackは、技術的には似た状態を作ります。

```text
HTLCを受け入れる
  ↓
preimageをすぐに出さない
  ↓
経路上のHTLCが未解決のまま残る
```

違いは目的です。

| 観点 | hodl invoice | Channel Jamming |
| --- | --- | --- |
| 目的 | 条件成立まで支払いを保留する | 経路上のリソースを占有する |
| 量 | 通常は必要な範囲 | 大量に作る |
| 時間 | アプリケーションに必要な短時間 | 長時間引き延ばす |
| 対象 | 自分のサービスの支払い | 他人のチャネルや経路 |
| 結果 | settleまたはcancelで解放する | 支払いを通りにくくする |

hodl invoiceは悪ではありません。むしろアプリケーションに必要な柔軟性を提供します。

ただし、「HTLCを保留できる」という同じ性質は、攻撃にも使えます。

## よくある誤解

### hodl invoiceは支払いを受け取った後に返金する仕組みである

厳密には違います。settleする前に保留しているため、支払いを確定してから返金するのではありません。

### hodl invoiceを使えば無期限に支払いを保留できる

できません。HTLCには期限があります。長く保留しすぎると、支払い失敗やオンチェーン解決のリスクが高まります。

### hodl invoiceはChannel Jammingそのものである

違います。hodl invoiceは正当な機能です。ただし、大量・長時間・攻撃目的で使うとjammingに似た効果を持ちます。

## まとめ

hodl invoiceは、Lightning支払いをすぐにsettleせず、アプリケーション条件が満たされるまで保留する仕組みです。

- 受取側はpreimageをすぐに出しません
- 支払いはacceptedのような保留状態になります
- 条件を満たせばsettleし、満たさなければcancelします
- 保留中は経路上のHTLC枠や流動性を使います
- 正当な機能ですが、保留という性質はChannel Jamming Attackの理解にもつながります

次の記事では、この「HTLCを保留できる」という性質が、どのようにDoS攻撃へ転じるのかを見ます。

## 参考資料

- [LND API: AddHoldInvoice](https://api.lightning.community/api/lnd/invoices/add-hold-invoice/index.html)（2026-06-29確認）
- [LND API: SettleInvoice](https://api.lightning.community/api/lnd/invoices/settle-invoice/index.html)（2026-06-29確認）
- [LND API: CancelInvoice](https://api.lightning.community/api/lnd/invoices/cancel-invoice/index.html)（2026-06-29確認）
- [BOLT #2: Peer Protocol](https://github.com/lightning/bolts/blob/master/02-peer-protocol.md)（2026-06-29確認）
- [Bitcoin Optech: Channel jamming attacks](https://bitcoinops.org/en/topics/channel-jamming-attacks/)（2026-06-29確認）

## 更新履歴

- 2026-06-29: 初版
