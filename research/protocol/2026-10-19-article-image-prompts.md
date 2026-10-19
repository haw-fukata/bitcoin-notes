# 記事冒頭画像の生成記録

生成日：2026-10-19
生成方法：組み込み image_gen ツール（各記事につき1回）。画像の文字・役割分担を目視確認済み。

## LN-GAP

保存先：`images/lightning-ln-gap/overview.png`

```text
Use case: infographic-diagram
Asset type: Japanese technical article opening overview, landscape 1536x1024.
Create one polished editorial explanatory infographic about LN-GAP, with large readable Japanese typography, generous whitespace, warm ivory background, navy text, teal normal flow and orange dispute flow. Flat illustrated chess board and two people, understated Bitcoin chain motif. No photos.
Exact title: 「LN-GAP：チャネル内で進め、争いだけをチェーンへ」
Main upper area: Alice and Bob playing chess within a rounded container labeled 「チャネル内」 with double arrow 「署名付きの手を交換」. Under board show 「通常：オフチェーンで進行・合意」.
Below and beside this, a committee of 5 simple member icons with clock and stamp, labeled 「venue」 and 「期限内の公開を証明」. Dashed attestations arrow from committee toward game messages; DO NOT depict committee holding coins or deciding chess legality.
Lower area: an orange arrow labeled 「不正・無応答で紛争」 leads from channel to Bitcoin chain container labeled 「Bitcoin上で解決」. Inside show exactly 3 sequential cards labeled 「主張」 → 「反論」 → 「反証」. Small secondary text 「必要な段階だけ実行」.
Bottom important visible footnote in full: 「venueの過半数が誠実という前提／研究段階の提案」
Keep text limited to quoted phrases plus Alice and Bob. No claims of unconditional trustlessness or production availability. Ensure logical flow unambiguous, crisp typography and legible at article width.
```

## Shielded Bitcoin

保存先：`images/bitcoin-shielded-bitcoin/overview.png`

```text
Use case: infographic-diagram
Asset type: Japanese technical article opening overview, landscape 1536x1024.
Create one polished editorial explanatory infographic for Shielded Bitcoin. Large clear Japanese type, ivory background, navy text, teal and restrained orange accents, flat illustrated wallets, sealed envelopes, blockchain and independent validators. Generous whitespace. Prioritize correct roles and readable few words.
Exact title: 「Shielded Bitcoin：内容を隠し、正しさを検証」
Top/main flow: wallet icon labeled 「送信者」 leads to a group of 3 cards labeled 「暗号化note」, 「nullifier」, 「ゼロ知識証明」. Under nullifier small label 「二重使用を検出」. Arrow goes to orange chain strip labeled 「Bitcoin：データの公開・順序付け」. Do NOT depict Bitcoin validating shielded proofs.
From this chain strip downward arrow labeled 「読み取る」 goes to teal box with 3 independent computer icons labeled 「indexer：証明を検証・状態を再構築」. Then right arrow to recipient wallet icon labeled 「受取人：自分のnoteを復号」.
Two neat side-by-side small bottom panels: 「隠す：内部送金の金額・相手」 with closed envelope icon, and 「見える：時刻・手数料・データ量」 with eye icon.
At very bottom a separate dashed orange boundary with readable text 「BTCの入出金は別設計／研究段階の提案」.
Maintain hierarchy: publication and verification are visibly different layers. No implication ordinary Bitcoin UTXOs themselves become encrypted. No promises of total anonymity or completed peg mechanism. No decorative tiny English filler.
```
