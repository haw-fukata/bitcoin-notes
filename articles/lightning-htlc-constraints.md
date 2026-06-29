---
title: "LightningのHTLCはなぜ詰まるのか — 数・金額・期限の制約を理解する"
emoji: "⚡"
type: "tech"
topics: ["bitcoin", "lightning", "security"]
published: true
---

## はじめに

Lightning Networkの支払いは、Bitcoinのオンチェーン取引よりも速く、少額決済にも向いています。では、Lightningの支払いは単なるメッセージのやり取りなのでしょうか。

答えは違います。

Lightningの支払いでは、チャネル上に**HTLC**という条件付きの支払いを一時的に追加します。HTLCはメモリ上の予約ではなく、チャネルのcommitment transactionに反映され、必要ならオンチェーンで解決できる状態です。

そのため、HTLCには数、金額、期限、手数料、dustなどの制約があります。この記事では、Channel Jamming Attackやhodl invoiceを理解する前提として、HTLCがなぜ有限リソースなのかを整理します。

この連載では、次の順番でLightningのHTLC保留とjammingを見ていきます。

1. HTLCの制約
2. [hodl invoiceとは何か](/articles/lightning-hodl-invoice)
3. [Channel Jamming Attackとは何か](/articles/lightning-channel-jamming-attack)

## 先に結論

HTLCの制約を一言でいうと、次のようになります。

- HTLCは未解決のまま無限に置けるわけではありません
- チャネルごとに未解決HTLC数の上限があります
- 未解決HTLCの合計金額にも上限があります
- HTLCには期限があり、中継ノードが安全に回収できるだけのCLTV差が必要です
- 小さすぎるHTLCはdustとして扱われ、オンチェーン解決時のリスクになります
- HTLCの追加や削除はcommitment transactionの状態更新として扱われます
- 中継ノードは、下流側HTLCが安全に片付く前に上流側HTLCを雑にfailできません

つまりHTLCは「軽い支払いメッセージ」ではなく、チャネルの有限な状態領域を使う条件付き支払いです。

## HTLCとは何か

HTLCは、Hashed Time Locked Contractの略です。日本語にすると「ハッシュと期限でロックされた契約」です。

Lightningの文脈では、ざっくり次の条件を持つ支払いです。

```text
ある期限までにpreimageを見せたら支払う
期限までにpreimageを見せなければ返金する
```

ここで出てくる要素は2つです。

- `payment_hash`: preimageをハッシュした値
- `preimage`: 支払いを成功させるための秘密値

受取人はpreimageを知っています。送金者と中継ノードは、最初はpreimageを知りません。受取人がpreimageを公開すると、その値が経路を逆向きに伝わり、各ホップのHTLCが順番に解決されます。

## 中継ノードでは何が起きているのか

中継ノードは、受け取ったHTLCをそのまま保管しているだけではありません。上流から受け取ったHTLCに対応して、下流へ新しいHTLCを出します。

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

Bから見ると、重要なのはこの対応関係です。

```text
Aから受け取るHTLC  <->  Cへ出すHTLC
```

Cがpreimageを出せば、BはCへ支払います。その代わり、同じpreimageを使ってAから回収します。

```text
Cがpreimageを出す
  ↓
BはCに支払う
  ↓
Bは同じpreimageでAから回収する
```

この対応関係が崩れると、中継ノードが損をします。後の章で詳しく見ます。

## HTLCはcommitment transactionに入る

Lightningのチャネルは、最新の残高状態を表すcommitment transactionを更新しながら進みます。HTLCを追加すると、そのHTLCもチャネル状態に反映されます。

つまりHTLCは、単なるルーティング用のメモではありません。最悪の場合、チャネルを閉じてオンチェーンで解決できる必要があります。

そのため、次のような制約が必要になります。

- commitment transactionが大きくなりすぎないこと
- オンチェーン手数料を払えること
- dust出力を増やしすぎないこと
- タイムロックに十分な余裕があること

この性質が、HTLCを有限リソースにしています。

## 未解決HTLC数の上限

チャネルには、同時に受け入れられる未解決HTLC数の上限があります。BOLT #2では、この値は`max_accepted_htlcs`として扱われ、通常のチャネルでは最大483です。

```text
未解決HTLC数 <= max_accepted_htlcs
```

この制限がないと、小さなHTLCを大量に追加してcommitment transactionや関連メッセージを大きくしすぎることができます。

ここがHTLC jammingの土台になります。攻撃者は大きな金額を使わなくても、小さなHTLCを大量に未解決のまま残すことで、HTLC枠を埋められます。

```text
チャネルのHTLC枠

[HTLC] [HTLC] [HTLC] ... [HTLC]
  1      2      3         483

→ 新しい支払いを追加しにくくなる
```

## 未解決HTLC合計額の上限

HTLCには数だけでなく、合計金額の制約もあります。これは`max_htlc_value_in_flight_msat`です。

```text
未解決HTLCの合計額 <= max_htlc_value_in_flight_msat
```

この制約は、相手に対してどれだけの金額を未解決状態で持たせるかを制限します。

Channel Jamming Attackのうち、**liquidity jamming**は主にこの性質を狙います。大きめのHTLCを保留することで、チャネルの使える流動性を拘束します。

## 最小HTLC金額

チャネルには`htlc_minimum_msat`もあります。これは、相手が受け入れるHTLCの最小金額です。

```text
HTLC金額 >= htlc_minimum_msat
```

極端に小さなHTLCを大量に送られると、ノードにとって処理コストのわりに得られる手数料が小さくなります。そのため、最小額を設定できます。

ただし、最小額を上げれば安全になる、という単純な話ではありません。最小額を上げすぎると、正当な少額支払いも通りにくくなります。

Lightningでは、スパム耐性と少額決済の使いやすさが常にトレードオフになります。

## commitment transactionの手数料

HTLCが増えると、commitment transactionやHTLC解決用トランザクションの重さも増えます。

Lightningでは、チャネルを閉じる必要が生じた場合でも、最新状態をオンチェーンで解決できなければいけません。そのため、手数料を払う側は、HTLCを追加したあとでも必要なオンチェーン手数料を払える状態を保つ必要があります。

ここで重要なのは、HTLCの追加が「オフチェーンだから無料」というわけではないことです。普段はオンチェーンに出ないだけで、いつでもオンチェーンに出せる形で設計されているため、手数料制約を無視できません。

## channel reserve

チャネルには、各参加者が最低限残しておくべき残高があります。これがchannel reserveです。

channel reserveは、相手が古い状態を不正に公開した場合などに、ペナルティを成立させるための経済的な余地として使われます。

HTLCを追加するときも、残高がreserveを下回らないようにする必要があります。

```text
残高 - HTLC - 手数料 >= channel reserve
```

実際の判定はもっと細かいですが、考え方としては「HTLCを追加しても、安全にチャネルを維持できる残高が必要」ということです。

## dust HTLC exposure

小さすぎるHTLCは、オンチェーン上で個別の出力として扱うと手数料に見合わないことがあります。このような出力はdustと呼ばれます。

HTLCがdustになる場合、commitment transaction上では個別のHTLC出力として載らず、実質的に手数料側へ寄ることがあります。

1つ1つは小さくても、大量にあると無視できません。そのため、BOLT #2には`max_dust_htlc_exposure_msat`という考え方があります。

HTLC jammingでは小額HTLCを大量に使うため、dust exposureの管理も重要になります。

## CLTV expiryと期限の余裕

HTLCには期限があります。Lightningでは、各ホップのHTLC期限に差をつけます。

```text
A → B → C

A→B のHTLC期限: 800
B→C のHTLC期限: 760
```

中継ノードBから見ると、下流のCに対するHTLCの期限のほうが早く、上流のAから受け取ったHTLCの期限のほうが遅い必要があります。

なぜでしょうか。

Cが期限ぎりぎりでpreimageを出した場合でも、Bはそのpreimageを使ってAから回収する時間が必要です。この余裕がなければ、BはCには支払ったのにAから回収できない、というリスクを負います。

この期限差が`cltv_expiry_delta`です。

## なぜHTLCは雑に消せないのか

ここが一番重要です。

中継ノードBが、Aから受け取ったHTLCをCへ転送しているとします。

```text
A -- incoming HTLC --> B -- outgoing HTLC --> C
```

Bは「この支払いは怪しい」と思ったとしても、Aへ先にfailを返してはいけません。B→Cのoutgoing HTLCがまだ安全に消えていない場合、Cがあとからpreimageを出してくる可能性があるからです。

危険な順序はこうです。

```text
1. A→B のincoming HTLCがある
2. B→C のoutgoing HTLCもある
3. BがAへ先にfailを返す
4. その後、Cがpreimageを出す
5. BはCへ支払う必要がある
6. しかしAからはもう回収できない
```

結果として、Bだけが損します。

```text
Aから回収できない
Cには支払う
→ Bが損する
```

このため、中継ノードは下流側HTLCの削除が安全にコミットされるまで、上流側HTLCをfailできません。

## 「安全にコミットされる」とは何か

Lightningのチャネル更新は、片方がメッセージを送った瞬間に完全確定するわけではありません。

単純化すると、更新は次のように進みます。

```text
update_add_htlc / update_fail_htlc / update_fulfill_htlc
  ↓
commitment_signed
  ↓
revoke_and_ack
  ↓
古い状態が失効し、新しい状態が安全になる
```

HTLCを追加する場合も、削除する場合も、両者のcommitment stateに反映され、古い状態が失効してはじめて安全になります。

そのため、「failメッセージを送ったからHTLCはもう消えた」とは言えません。対応する状態更新が安全に確定している必要があります。

## HTLCの制約まとめ

HTLCの主な制約を表にすると、次のようになります。

| 制約 | 何を制限するか | 関係する問題 |
| --- | --- | --- |
| `max_accepted_htlcs` | 未解決HTLC数 | HTLC jamming |
| `max_htlc_value_in_flight_msat` | 未解決HTLCの合計額 | liquidity jamming |
| `htlc_minimum_msat` | 最小HTLC金額 | 小額スパムと少額決済のトレードオフ |
| commitment fee | オンチェーン解決コスト | force close時の安全性 |
| channel reserve | チャネル維持に必要な残高 | 不正時のペナルティ余地 |
| dust exposure | dust HTLCの合計リスク | 小額HTLCの大量発生 |
| CLTV expiry | HTLCの期限 | 中継ノードの回収時間 |

## よくある誤解

### HTLCはメモリ上のキューである

違います。HTLCはチャネル状態に入り、必要ならオンチェーンで解決される条件付き支払いです。

### 怪しいHTLCはすぐfailすればいい

中継ノードは、下流側HTLCが安全に片付く前に上流側をfailすると、自分だけ損する可能性があります。

### HTLC jammingには大きな資金が必要である

必ずしもそうではありません。HTLC枠を狙う場合、小額HTLCでも攻撃の材料になります。

## まとめ

HTLCはLightning支払いの中心ですが、無制限に扱えるものではありません。

- HTLCには数、金額、期限、手数料、dustなどの制約があります
- HTLCはcommitment transactionに反映されるため、オンチェーン解決可能性を考える必要があります
- 中継ノードはincoming HTLCとoutgoing HTLCの対応関係を守らなければ損をします
- この制約が、hodl invoiceの挙動やChannel Jamming Attackを理解する土台になります

次の記事では、HTLCを正当に保留する仕組みとして、hodl invoiceを見ていきます。

## 参考資料

- [BOLT #2: Peer Protocol](https://github.com/lightning/bolts/blob/master/02-peer-protocol.md)（2026-06-29確認）
- [BOLT #3: Bitcoin Transaction and Script Formats](https://github.com/lightning/bolts/blob/master/03-transactions.md)（2026-06-29確認）
- [BOLT #4: Onion Routing Protocol](https://github.com/lightning/bolts/blob/master/04-onion-routing.md)（2026-06-29確認）
- [Bitcoin Optech: Channel jamming attacks](https://bitcoinops.org/en/topics/channel-jamming-attacks/)（2026-06-29確認）

## 更新履歴

- 2026-06-29: 初版
