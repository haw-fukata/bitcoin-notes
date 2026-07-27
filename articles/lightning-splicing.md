---
title: "LightningのSplicingとは何か — チャネルを閉じずに資金を出し入れする仕組み"
emoji: "✂️"
type: "tech"
topics: ["bitcoin", "lightning", "splicing"]
published: true
---

## はじめに

Lightningチャネルへ資金を追加したい、またはチャネル内の資金をオンチェーンへ戻したい場合、チャネルを一度閉じて開き直す方法があります。

しかし、それでは閉鎖と再開設のオンチェーントランザクションが必要になり、新しいチャネルが使えるまで支払い経路も途切れます。

**Splicing**は、既存チャネルを論理的に継続したまま、チャネル容量を増減させる仕組みです。

この記事では、まず資金の動きを確認し、そのあとでfunding outputの置き換えと、承認待ち中もチャネルを安全に使う仕組みを見ていきます。

## 先に結論

Splicingの要点は、次のとおりです。

- splice-inでは、オンチェーン資金を既存チャネルへ追加できます
- splice-outでは、チャネル内の自分の残高をオンチェーンへ取り出せます
- 現在のfunding outputを使い、新しいfunding outputを作ります
- チャネル相手との関係やオフチェーン状態は継続します
- 交渉中は一時的にチャネル更新を止めます
- splice transactionへの署名後は、オンチェーン承認を待ちながら支払いを再開できます
- Splicingには相手の協力とオンチェーン手数料が必要です

Splicingはチャネル残高を直接書き換える機能ではありません。**現在のfunding outputを、容量の異なる新しいfunding outputへ置き換える仕組み**です。

## Splicingの全体像

Lightningチャネルの資金は、オンチェーンのfunding outputにロックされています。

```text
Funding Transaction 1
  └─ Funding Output: 1,000,000 sat
       └─ AliceとBobのチャネル
```

Splicingでは、古いfunding outputを入力にしたsplice transactionを作り、新しいfunding outputへ資金を移します。

```text
古いFunding Output
  ↓ Splice Transactionで使用
新しいFunding Output
  ↓
容量を変更したチャネル
```

オンチェーン上ではfunding outputが変わりますが、AliceとBobのチャネルは論理的に継続します。

## Splice-inで資金を追加する

AliceとBobが、次のチャネルを持っているとします。

```text
チャネル容量: 1,000,000 sat

Alice [600,000 sat] ───── [400,000 sat] Bob
```

Aliceが自分のオンチェーンUTXOから500,000 satを追加します。

```text
入力:
  古いFunding Output  1,000,000 sat
  AliceのUTXO           500,000 sat

出力:
  新しいFunding Output 1,500,000 sat
```

新しいチャネル状態は、概念的に次のようになります。

```text
チャネル容量: 1,500,000 sat

Alice [1,100,000 sat] ───── [400,000 sat] Bob
```

Bobの残高は変わらず、Aliceが追加した500,000 satはAlice側の残高になります。

これが**splice-in**です。既存のpeerやチャネル状態を維持したまま、チャネル容量と自分の送金可能額を増やせます。

実際には、ここからsplice transactionのマイニング手数料などが差し引かれます。

## Splice-outで資金を取り出す

同じ1,000,000 satのチャネルから、Aliceが200,000 satをオンチェーンへ取り出す場合を考えます。

```text
入力:
  古いFunding Output 1,000,000 sat

出力:
  新しいFunding Output 800,000 sat
  Aliceのアドレス      200,000 sat
```

新しいチャネル状態は、概念的に次のようになります。

```text
チャネル容量: 800,000 sat

Alice [400,000 sat] ───── [400,000 sat] Bob
```

これが**splice-out**です。

Aliceはチャネル全体を閉じず、自分の残高の一部だけをオンチェーンへ戻せます。出力先に支払先のBitcoinアドレスを指定すれば、チャネル残高を使ったオンチェーン支払いとしても利用できます。

ただし、取り出せるのは自分側の残高です。相手の残高を取り出すことはできず、取り出した後もchannel reserveなどの条件を満たす必要があります。

## 閉鎖して再オープンする場合との違い

Splicingを使わずに容量を変更する場合、概念的には次の処理になります。

```text
既存チャネルを閉じる
  ↓
オンチェーン資金を回収
  ↓
新しいFunding Transactionを作る
  ↓
新しいチャネルを開く
```

Splicingでは、古いfunding outputから新しいfunding outputへの移行を1つのsplice transactionで行います。

| 項目 | 閉鎖して再オープン | Splicing |
| --- | --- | --- |
| オンチェーン処理 | 閉鎖と再開設 | funding outputの置き換え |
| 論理的なチャネル | 一度終了する | 継続する |
| 支払い停止 | 長くなりやすい | 主に交渉中だけ |
| チャネル状態 | 新しく作り直す | 既存状態を引き継ぐ |
| 容量変更 | 新しいチャネルで設定 | splice-in / splice-out |

## Splicingの処理

現行BOLT #2では、次のように処理します。

```text
チャネル更新を一時停止
  ↓
増減させる金額を合意
  ↓
splice transactionを共同構築
  ↓
新しいcommitment transactionを交換
  ↓
splice transactionへ署名
  ↓
チャネル更新を再開
  ↓
オンチェーン承認後に完全移行
```

### 1. Channel Quiescence

通常のチャネルでは、HTLCの追加や削除によって残高が変化しています。その途中でspliceを始めると、両者が異なる残高を新しいfunding outputへ引き継ぐ可能性があります。

そこで、最初に`stfu`メッセージを交換し、チャネルをquiescent状態にします。

```text
Alice                    Bob
  |------ stfu ---------->|
  |<--------- stfu -------|
  |                       |
  |  新しいupdateを停止    |
```

quiescence中は新しいチャネル更新を送らず、AliceとBobが同じcommitment stateを持つ状態にします。

### 2. 金額を提示する

開始側は`splice_init`を送り、相手は`splice_ack`で応答します。

両者は`funding_contribution_satoshis`で、自分の残高をどれだけ増減させるかを示します。

```text
正の値: splice-in
負の値: splice-out
ゼロ:   自分の残高は変更しない
```

仕様上は、AliceとBobの両方が同じsplice transactionへ資金を追加したり、資金を取り出したりできます。

### 3. Splice Transactionを共同構築する

両者はInteractive Transaction Constructionを使い、`tx_add_input`や`tx_add_output`を交換してtransactionを組み立てます。

```text
入力:
  現在のFunding Output
  必要ならAliceやBobの追加UTXO

出力:
  新しいFunding Output
  必要ならsplice-out先やchange
```

新しいチャネル容量は、概念的に次のようになります。

```text
新しい容量
  = 古い容量
  + Aliceのfunding contribution
  + Bobのfunding contribution
```

### 4. 新しいCommitment Transactionを作る

splice transactionへ署名する前に、両者は新しいfunding outputを使用するcommitment transactionを作り、署名を交換します。

```text
Splice Transaction
  └─ 新しいFunding Output
       └─ 新しいCommitment Transaction
```

これにより、splice transactionがブロードキャストされたあと相手が協力しなくなっても、各自が新しいfunding outputから自分の残高を回収できます。

## 承認待ち中も支払いを続けられる

splice transactionへの署名が完了するとquiescenceは終了し、オンチェーン承認を待ちながらチャネル更新を再開できます。

ただし、承認前は複数の可能性が残っています。

```text
古いFunding Output
  ├─ 古いCommitment Transaction
  ├─ Splice候補 → Commitment Transaction
  └─ RBF候補    → Commitment Transaction
```

どのtransactionが最終的に承認されても安全なように、ノードは各候補に対応するcommitment transactionを管理します。新しいHTLCも、すべての有効な候補で成立する必要があります。

手数料が低くて承認されない場合は、RBFでより高い手数料のsplice transactionを作れます。

いずれかの候補が必要な承認深度へ達すると、両者は`splice_locked`を交換します。両者が同じtransactionを指定したら、そのfunding outputへ完全に移行し、古い候補を破棄できます。

## 公開チャネルのSCIDは変わる

SCIDは、funding transactionが承認されたブロック位置をもとに作られます。そのため、Splicingでfunding transactionが変わると、公開チャネルのSCIDも変わります。

両者は新しいsplice transactionについて`channel_announcement`を作り直します。またBOLT #7は、以前のSCIDを使った支払いもしばらく中継するよう推奨しています。

論理的なチャネルは継続しますが、オンチェーン上のfunding outpointや公開SCIDまで同じままではありません。

## 制約

### オンチェーン手数料が必要

Splicingはfunding outputを置き換えるため、オンチェーントランザクションとマイニング手数料が必要です。

### 相手の協力が必要

現在のfunding outputは2者の署名で使用するため、Splicingには相手の協力が必要です。相手がオフラインまたは拒否した場合は実行できません。

### 交渉中は支払いを一時停止する

オンチェーン承認を待つ間ずっと停止するわけではありませんが、quiescenceからsplice transactionへの署名が終わるまでは、新しいチャネル更新を止めます。

### 実装の状態管理が複雑になる

承認前は、古いfunding outputと複数のsplice・RBF候補を同時に管理します。再接続やreorgも含め、両者が同じ候補とcommitment stateを保持する必要があります。

## よくある誤解

### チャネル残高だけをオフチェーンで変更する

違います。古いfunding outputを使用し、新しいfunding outputを作るオンチェーン処理です。

### Splice-inした資金はすぐすべて使える

承認待ち中の支払いは、古いfunding outputを含む有効な候補すべてで成立する必要があります。追加した容量を完全に利用できるのは、通常は新しいfunding outputへ移行したあとです。

### Splice-outなら相手の残高も取り出せる

取り出せるのは自分側の残高です。相手の残高やchannel reserveを侵害するspliceは拒否されます。

### チャネルのオンチェーン情報も変わらない

論理的なチャネルは継続しますが、funding outpointは変わります。公開チャネルではSCIDも更新されます。

## 仕様と実装状況

Splicing仕様は2026年3月23日にLightning BOLTsへマージされ、2026年7月27日時点でBOLT #2とBOLT #7に含まれています。

EclairやCore Lightningでは仕様策定中から実装と相互運用が進められ、Phoenixも公式Splicingメッセージを使うリリースを公開しています。一方で、対応範囲は実装やバージョンによって異なります。

## まとめ

Splicingは、既存チャネルのfunding outputを新しいfunding outputへ置き換え、チャネル容量を増減させる仕組みです。

- splice-inでオンチェーン資金を追加できます
- splice-outで自分の残高をオンチェーンへ取り出せます
- チャネルを閉じて開き直す必要がありません
- 交渉中はquiescenceで状態を固定します
- 新しいcommitment transactionを用意してからsplice transactionへ署名します
- 署名後は、承認を待ちながら支払いを再開できます
- 承認前は複数のfunding候補とcommitmentを管理します
- オンチェーン手数料と相手の協力が必要です

Splicingによって、Lightningチャネルは「一度決めた容量を閉鎖まで使い続けるもの」から、運用中にオンチェーン残高と資金を出し入れできるものへ変わります。

## 参考資料

- [BOLT #2: Peer Protocol for Channel Management](https://github.com/lightning/bolts/blob/master/02-peer-protocol.md)（2026-07-27確認）
- [BOLT #7: P2P Node and Channel Discovery](https://github.com/lightning/bolts/blob/master/07-routing-gossip.md)（2026-07-27確認）
- [lightning/bolts#1160: Channel Splicing](https://github.com/lightning/bolts/pull/1160)（2026-07-27確認）
- [lightningdevkit/rust-lightning#1621: Dual-funded channels and Splicing Project Tracking](https://github.com/lightningdevkit/rust-lightning/issues/1621)（2026-07-27確認）
- [ACINQ Phoenix Releases](https://github.com/ACINQ/phoenix/releases)（2026-07-27確認）

## 更新履歴

- 2026-07-27: 初版
