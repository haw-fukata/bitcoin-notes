---
title: "Utreexo入門 — UTXOセットをMerkle forestのルートに置き換える"
emoji: "🌲"
type: "tech"
topics: ["bitcoin", "utreexo", "blockchain", "merkle"]
published: true
---

## はじめに

Bitcoinのフルノードは、過去のトランザクションを検証するだけでなく、「現在どのoutputを使用できるか」を示すUTXOセットを管理します。

UTXOセットは、新しいブロックを検証するときに頻繁に読み書きされます。ノードは各inputが本当に存在するUTXOを参照しているか、すでに使用されていないかを確認し、ブロックの新しいoutputをUTXOセットへ追加します。

**Utreexo**は、このUTXOセットを動的なMerkle accumulatorで表し、検証ノードが木全体ではなく少数のルートだけを保持できるようにする仕組みです。

ストレージとランダムディスクI/Oを減らせる一方、inputが有効なUTXOを参照することを示すproofが必要になり、通信量は増えます。

さらに2026年9月には、SwiftSyncのhints fileと**implicit deletion**を組み合わせ、Initial Block Download（IBD）中のproof帯域を大きく減らす提案が公開されました。

この記事では、UTXOセットの役割からUtreexoのMerkle forest、proverとCompact State Node、そして最新のIBD改善提案まで順に説明します。

:::message
UtreexoはBitcoinのトランザクション履歴を圧縮する仕組みではありません。ノードが「現在使用できるコイン」を確認するために持つUTXO状態を、小さな暗号学的commitmentで表す仕組みです。
:::

## 先に結論

Utreexoの要点は次のとおりです。

- UTXOセットは、現在使用可能なtransaction outputの集合です
- 通常のフルノードは、UTXOセットをローカルのkey-value databaseとして保持します
- Utreexoは、UTXOを複数の完全Merkle treeからなる動的なaccumulatorで表現します
- Compact State Nodeは、木全体ではなく`O(log n)`個のルートだけを保持できます
- その代わり、各inputについてUTXOの存在を示すMerkle proofを受け取る必要があります
- proofを生成するproverは、Merkle forest全体または必要な内部nodeを保持します
- Utreexoは検証を省略するのではなく、ローカルdatabase参照をcommitmentとmembership proofの検証へ置き換えます
- 従来方式のUtreexo IBDではproofの追加帯域が大きくなる問題があります
- 最新のimplicit deletion提案は、IBD中だけ将来使用済みになるTXOを最初からforestへ残さず、明示的な削除proofをほぼ不要にします
- BIP 181〜183は2026年9月28日時点でDraftであり、仕様と実装は変更される可能性があります

## UTXOセットとは何か

Bitcoinでは、アカウントごとの残高を直接記録しません。トランザクションが作ったoutputを、別のトランザクションのinputが使用します。

まだ使用されていないoutputを**UTXO（Unspent Transaction Output）**と呼び、その集合がUTXOセットです。

```text
Transaction A
  └─ Output 0: 50,000 sat  ← 未使用なのでUTXO

Transaction B
  ├─ Input: AのOutput 0を使用
  ├─ Output 0: 30,000 sat  ← 新しいUTXO
  └─ Output 1: 19,500 sat  ← 新しいUTXO
```

Transaction Bが有効か確認するには、少なくとも次の点を調べます。

- AのOutput 0が実在するか
- AのOutput 0がまだ使用されていないか
- 提示されたscriptとwitnessがspending conditionを満たすか
- 入力額より不正に多い出力を作っていないか

通常のフルノードは、最初の2点を効率よく確認するためにUTXOセットをローカルへ保存します。

ブロックを受け取るたび、使用されたUTXOを削除し、新しく作られたUTXOを追加します。

```text
現在のUTXOセット
  ↓ blockを検証
使用されたUTXOを削除
新しいUTXOを追加
  ↓
次のUTXOセット
```

## ブロック履歴とUTXOセットは別物

Utreexoを理解するとき、ブロックチェーン履歴とUTXOセットを分けて考える必要があります。

| データ | 内容 | 主な用途 |
| --- | --- | --- |
| ブロック履歴 | 過去に発生した全トランザクション | 過去から現在までの状態遷移を検証する |
| UTXOセット | 現在使用可能なoutput | 新しいinputの存在と未使用状態を確認する |

フルノードは、全ブロックを最初から検証して現在のUTXOセットを構築できます。pruned nodeは、検証済みの古いブロック本体を削除できますが、現在のUTXOセットは継続して必要です。

Utreexoが小さくする対象は、主にこの**現在状態**です。過去のブロックを数百バイトに圧縮するわけではありません。

## Utreexoの基本アイデア

Utreexoは、UTXOをhash化したleafとしてMerkle treeへ入れます。ただし、常に1本の完全な木を維持するのではなく、leaf数を2進数で表したときに対応する複数の完全Merkle tree、つまり**Merkle forest**を使います。

たとえば10個のleafは、次のように分けられます。

```text
10 = 8 + 2
```

したがって、8 leafの木と2 leafの木からなるforestになります。

```text
             Root A                     Root B
          /          \                 /      \
       ...            ...            L8        L9
      /   \          /   \
    L0    L1       ...    L7

      8 leafの完全木                  2 leafの完全木
```

すべての内部nodeを保持すれば、任意のleafに対するMerkle proofを作れます。しかし、UTXOの存在を検証するだけなら、現在のrootと対象leafからrootまでの兄弟hashがあれば十分です。

## Compact State Nodeとprover

Utreexoでは、役割を大きく2つに分けて考えられます。

### Compact State Node

Compact State Node（CSN）は、Merkle forestのrootだけを保持します。

```text
保持するもの:
  Root A
  Root B
  leaf数などの状態

保持しないもの:
  全leaf
  全内部node
```

rootの数はleaf数に対して`O(log n)`です。leafが増えてもroot数は非常にゆっくり増えます。

CSNは、渡されたleaf dataとMerkle proofを検証できます。しかし木の下部を持たないため、任意のleafのproofを自分で生成することはできません。

### Prover

proverは、proofを生成するためのforestデータを保持します。

```text
prover
  ├─ leaf
  ├─ 内部node
  └─ 任意のUTXOに対するproofを生成

CSN
  ├─ root
  └─ 受け取ったproofを検証
```

Utreexoは「すべてのノードが数百バイトしか持たなくなる」設計ではありません。proofを提供する役割には、より多くの状態と帯域が必要です。

利用者が自分のUTXOに関するproofを保持し、状態更新に合わせてproofを更新する構成も考えられます。どこがproof維持コストを負担するかは、Utreexoの重要な設計上の論点です。

## Merkle proofでUTXOの存在を確認する

4つのleafを持つ単純なMerkle treeで、`L1`の存在を確認するとします。

```text
             Root
            /    \
          H01    H23
         /  \    /  \
       L0   L1  L2  L3
```

検証者が`Root`だけを持っている場合、`L1`に加えて次の兄弟hashを受け取ります。

```text
proof = [L0, H23]
```

検証者は次のようにrootを再計算します。

```text
H01  = H(L0 || L1)
Root'= H(H01 || H23)
```

`Root'`が手元の`Root`と一致すれば、`L1`がそのaccumulatorへ含まれていることを確認できます。

Bitcoin用のUtreexo proofでは、leaf dataからUTXO hashを計算し、proof内のhashと位置情報を使ってrootへ到達します。BIP 182のDraftでは、通常ノードがローカルUTXO databaseから取得する情報を、leaf dataとaccumulator proofから得る形式を定義しています。

## 新しいUTXOを追加する

Utreexoのforestは、leaf数の2進表現に対応しています。新しいleafを1つ追加するとき、同じ大きさの木があれば結合し、空いている階層まで繰り上げます。

これは2進数の加算に似ています。

```text
既存: 5 leaf = 4 + 1
追加: 1 leaf
結果: 6 leaf = 4 + 2
```

最下位の1-leaf treeと新しいleafをhashし、2-leaf treeを作ります。

```text
Root(1 leaf) + New leaf
          ↓ hash
      Root(2 leaves)
```

その階層がすでに埋まっていれば、さらに上の階層へ繰り上げます。

追加操作は現在のrootだけで進められます。木全体をディスクからランダムreadする必要がなく、CSNにとって効率的です。

## UTXOを削除する

使用済みになったUTXOは、accumulatorから削除する必要があります。

概念的には、削除対象のleafとその親を取り除き、兄弟nodeを親の位置へ持ち上げます。その後、上位のhashを再計算します。

```text
削除前:

        Parent
        /    \
       A      B  ← 削除

削除後:

        A  ← 上へ移動
```

ただし、rootしか持たないCSNは、削除対象の兄弟や上位経路を知りません。そのため、削除後のrootを計算するにはMerkle proofが必要です。

通常のUtreexo block validationでは、ブロック内で使用される複数UTXOをまとめたbatched proofを使います。複数proofで共通する内部hashを一度だけ送れるため、個別proofを並べるより小さくできます。

## Utreexoは何を節約し、何を増やすのか

通常ノードとUtreexo CSNの違いを単純化すると次のようになります。

| 資源 | 通常ノード | Utreexo CSN |
| --- | --- | --- |
| UTXO状態の保存 | UTXO database全体 | 少数のforest root |
| input確認 | ローカルdatabaseを参照 | leaf dataとproofを検証 |
| ランダムディスクI/O | 必要 | 大幅に削減可能 |
| block受信帯域 | 通常のblock | blockに加えてproof |
| proof生成 | 不要 | CSN単体ではできない |

Utreexoはコストを消すのではなく、場所を移します。

```text
ローカルストレージとdisk I/O
             ↓
proofの生成、転送、hash検証
```

小型端末やストレージの遅い環境では有利になり得ます。一方、帯域が限られる環境や、proofを提供するpeerが少ない環境では不利になる可能性があります。

## コンセンサス変更は必要なのか

Utreexo BIP 181〜183のDraftは、現在のBitcoinコンセンサスと同じ有効なchain tipを検証する、コンセンサス互換な方式として提案されています。

Utreexo nodeは、従来nodeとは異なる内部表現を使いますが、有効と判断するBitcoin transactionとblockのルールを変えません。その意味で、Utreexo nodeを動かすためにBitcoinのsoft forkを必須とはしません。

ただし、既存のBitcoin P2PメッセージにはUtreexo proofが含まれません。proof付きblockやtransactionを交換するためには、Utreexo対応node間のP2P拡張が必要です。BIP 183 Draftはこの通信形式を扱います。

```text
コンセンサスルール:
  既存Bitcoinと同じ

内部状態表現:
  UTXO database → Utreexo roots

P2P通信:
  proofを交換する拡張が必要
```

## IBDではproof帯域が大きな問題になる

新しいnodeがgenesis blockから現在までを検証する処理をInitial Block Download（IBD）と呼びます。

CSNが従来方式でIBDを行う場合、各blockで使用されるTXOのbatched inclusion proofを受け取る必要があります。

2026年9月の提案では、最適化なしの場合、Utreexo proofの平均的な追加サイズはblock本体と同程度とされます。現在までのchainに対して、合計ダウンロード量は約1.3 TBになるという試算です。

最近作られたUTXOは比較的早く使用される傾向があります。そのため、最近のleafやproofをRAMへcacheすれば、必要データの多くを再利用できます。それでも明示的な削除proofが約200 GB残るという試算が示されています。

```text
Utreexoの目的:
  ローカルUTXO状態を小さくする

IBDで生じる問題:
  過去の各時点で削除proofを受け取るため帯域が増える
```

ここで登場するのが、SwiftSyncとimplicit deletionです。

## SwiftSyncのhints file

IBDでは、同期先となるblock heightが決まっています。そのheightまでに、過去に作られた各TXOが使用済みになるか、まだUTXOとして残るかを事前に知ることができます。

SwiftSyncは、この情報を**hints file**としてnodeへ渡します。

```text
TXO A → 同期先までに使用済み
TXO B → 同期先でも未使用
TXO C → 同期先までに使用済み
```

「外部から渡されたファイルを信用してよいのか」という疑問が生じます。

SwiftSyncの構想では、使用済みとされたoutputと、それを使用するinputをhash aggregateへ加減算し、IBD終了時にaggregateがゼロになることを確認します。

```text
作成されたspent TXOをaggregateへ加える
対応するinputをaggregateから引く
  ↓
最終的にゼロなら対応が取れている
```

Bitcoin OptechはSwiftSyncを、untrustedなhints fileを使いつつ、最終aggregateで整合性を確認する方式と説明しています。

ただし、assumevalidを使用する実装と、全scriptを検証する非assumevalid版では前提が異なります。2026年9月時点では、Florestaのassumevalid実装と非assumevalid方式の開発が続いています。

## implicit deletionとは何か

通常のUtreexoでは、TXOを一度forestへ追加し、将来使用された時点でproofを使って削除します。

```text
TXOを追加
  ↓
数block後に使用
  ↓
proofを受け取って削除
```

IBDではhints fileにより、そのTXOが同期先までに使用済みになることを先に知っています。そこで、追加操作の時点で将来の削除を織り込めないか、というのがimplicit deletionの発想です。

2つのleaf `A`と`B`を追加し、`B`が同期先までに使用済みになる場合を考えます。

通常なら、いったん親を作ります。

```text
       H(A || B)
       /       \
      A         B
```

その後Bを削除すると、兄弟Aが上へ移動します。

```text
       A
```

implicit deletionでは、Bを追加する時点で使用済みになると知っているため、`H(A || B)`を作らず、最初からAを上の階層へ移動します。

```text
通常:
  AとBをhash → 後からBをproof付きで削除 → Aが上へ

implicit deletion:
  Bが使用済みと知っている → Aを直接上へ
```

このルールをforest全体の追加処理へ組み込むと、明示的な削除proofを受け取らなくても、同期先で通常の削除をすべて適用した場合と同じroot setへ到達できます。

## なぜIBD中だけ使えるのか

implicit deletionには、「そのTXOが将来使用される」と事前に知っている必要があります。

IBDでは同期先heightまでのblockがすでに存在するため、hints fileを作れます。しかしchain tipへ追いついた後は、新しいUTXOが将来いつ使用されるか分かりません。

したがって、implicit deletionが使える範囲は次のようになります。

```text
genesis ───────── 同期先height ───── 新しいblock
       implicit deletion使用可能       通常proofが必要
```

IBD完了後のsteady stateでは、通常のUtreexoと同様にproofを受け取り、inputの存在確認と削除を行います。

この改善は、Utreexoからproofを完全になくすものではありません。**過去の履歴を同期する区間に限って、明示的な削除proofをほぼ不要にする提案**です。

## 並列処理しやすくなる理由

通常のIBDでは、前のblockでUTXOセットを更新してから、次のblockを処理するという逐次依存が強くあります。

SwiftSyncは、同期先で使用済みになるoutputをhintsから把握し、hash aggregateで最終整合性を確認します。そのため、検証処理の一部をblock到着順に縛られず進められます。

implicit deletionの提案者は、assumevalid版の開発中実装について、blockを受信した順に処理でき、安価なRaspberry PiでもCPUよりネットワーク帯域がボトルネックになったと報告しています。

ただし、Merkle forestの追加順序は最終rootに影響します。accumulator stateの適用には依然として順序が必要で、提案では小さな逐次処理threadが、適用可能になった変更を順番に反映します。

## Utreexoは検証を弱くするのか

通常のUtreexo operationでは、CSNはUTXO databaseを持たない代わりに、次を検証します。

- leaf dataが参照するUTXOを正しく表すか
- Merkle proofが現在のrootへ到達するか
- blockとtransactionが既存のBitcoin consensus ruleを満たすか
- 削除と追加によって新しいrootが正しく計算されるか

したがって、Utreexoは「UTXOの存在確認を行わない方式」ではありません。

```text
通常node:
  ローカルdatabaseにUTXOがあることを確認

Utreexo CSN:
  cryptographic proofがrootに一致することを確認
```

一方、実装の安全性が自動的に保証されるわけではありません。leaf encoding、position計算、追加順序、batched proof、root更新のいずれかに不整合があれば、node間でstateが分かれる可能性があります。

BIPと実装がまだDraft・開発段階であることは、運用時に重要な前提です。

## 通常nodeとの役割分担

Utreexoが普及しても、すべてのnodeがCSNになる必要はありません。

```text
Utreexo prover
  ├─ forestデータを保持
  ├─ proofを生成
  └─ proof付きblock / TXを配信
          ↓
Compact State Node
  ├─ rootだけを保持
  ├─ proofを検証
  └─ Bitcoin consensus ruleを検証
```

proverが存在しなければ、CSNは必要なproofを取得できません。逆にproverを一者だけに依存すると、可用性やプライバシーの懸念が生じます。

Utreexoの評価では、「CSNがどれだけ小さくなるか」だけでなく、「誰がproofを作り、更新し、配るのか」を含むネットワーク全体のコストを見る必要があります。

## よくある誤解

### 「ブロックチェーン全体が数百バイトになる」

なりません。小さくなるのは、主にUTXOセットを表すaccumulator stateです。過去のblockを検証するための履歴、block header、wallet dataなどは別に必要です。

### 「Merkle rootだけを信用する軽量walletである」

Utreexo CSNは、rootを使いながらblockとtransactionを検証するfully validating nodeを目指す設計です。headerだけを確認するSPVとは異なります。

### 「proofがあるのでproverを信用する必要がない」

正しいproofでなければrootと一致しないため、proverに正しさを委ねる必要はありません。ただし、proofを渡さない、特定のtransactionを隠すといった可用性上の攻撃は別問題です。

### 「Utreexoにはsoft forkが必要」

Draft BIPは、既存Bitcoin consensusと互換なnode実装として提案されています。ただしproof交換用のP2P拡張と、Utreexo対応peerは必要です。

### 「implicit deletionで通常時のproofも不要になる」

不要になるのは、同期先までのspent情報を事前に持てるIBD区間です。chain tipへ追いついた後は、将来のspent状態が分からないため通常のproofへ戻ります。

## 制約・未解決点

- BIP 181、182、183は2026年9月28日時点でDraftです
- BIP番号、wire format、leaf encoding、更新アルゴリズムは変更される可能性があります
- implicit deletionと非assumevalid SwiftSyncの実装は開発中です
- 約1.3 TB、約200 GBという値は提案者による現在のchainに対する試算であり、blockchainの成長やcache条件で変わります
- Optechが紹介した「near-zero proof overhead」は2026年9月の新提案についての見通しです。既存リリースの達成値と区別する必要があります
- proofを提供するproverの分散性、可用性、帯域負担、query privacyは引き続き検討が必要です
- 悪意あるhints fileが無効blockを有効にできずDoSにしかならないという主張は、完成した非assumevalid実装で追加検証する必要があります
- 本記事では、小規模forestによるアルゴリズム再現や実装ベンチマークは行っていません

## まとめ

Utreexoは、Bitcoin nodeが保持するUTXOセットを、動的なMerkle forestのrootへ置き換える仕組みです。

Compact State Nodeは少数のrootだけを保持し、inputごとに提供されるleaf dataとMerkle proofを使ってUTXOの存在を確認します。これによりストレージとランダムdisk I/Oを大幅に減らせる一方、proofの生成と転送という新しいコストが生じます。

従来のUtreexo IBDでは、過去の各blockで削除proofを受け取る帯域負担が大きな問題でした。2026年9月のimplicit deletion提案は、SwiftSyncのhints fileから同期先までのspent情報を得て、追加時点で将来の削除を反映します。

これによりIBD中の明示的な削除proofをほぼ不要にできる可能性があります。ただし、IBD完了後は通常のproof検証へ戻り、BIPと実装はいずれもまだ開発段階です。

Utreexoの本質は、検証を省略することではありません。**大きなローカル状態を保持する代わりに、小さなcommitmentと、その状態に含まれることを示すproofを検証する**というコストの置き換えです。

## 参考資料

- [Utreexo: A dynamic hash-based accumulator optimized for the Bitcoin UTXO set](https://eprint.iacr.org/2019/611)（2026-09-28確認）
- [Utreexo project overview — MIT Media Lab](https://www.media.mit.edu/projects/utreexo/overview/)（2026-09-28確認）
- [BIP 181, 182, 183: BIPs for Utreexo — bitcoin/bips PR #1923](https://github.com/bitcoin/bips/pull/1923)（2026-09-28確認）
- [BIP 181 Draft: Utreexo accumulator specification](https://github.com/utreexo/biptreexo/blob/main/bip-0181.md)（2026-09-28確認）
- [BIP 182 Draft: Utreexo validation](https://github.com/utreexo/biptreexo/blob/main/bip-0182.md)（2026-09-28確認）
- [BIP 183 Draft: Utreexo P2P messages](https://github.com/utreexo/biptreexo/blob/main/bip-0183.md)（2026-09-28確認）
- [Implicit Deletions and Improvements in Utreexo IBD](https://delvingbitcoin.org/t/implicit-deletions-and-improvements-in-utreexo-ibd/2881)（2026-09-28確認）
- [SwiftSync - smarter synchronization with hints](https://gist.github.com/RubenSomsen/a61a37d14182ccd78760e477c78133cd)（2026-09-28確認）
- [Bitcoin Optech Newsletter #423](https://bitcoinops.org/ja/newsletters/2026/09/18/)（2026-09-28確認）
- [utreexod](https://github.com/utreexo/utreexod)（2026-09-28確認）
- [Floresta PR #1115: Assume-Valid SwiftSync with accumulator building](https://github.com/getfloresta/Floresta/pull/1115)（2026-09-28確認）

## 更新履歴

- 2026-09-28: 初版ドラフト
