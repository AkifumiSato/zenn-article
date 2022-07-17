---
title: "Next.jsはどうやってスクロール位置を復元するのか"
emoji: "📜"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

[Next.js](https://nextjs.org/)にはexperimental(実験的機能)で`scrollRestoration`というフラグが存在します。デフォルトでもSSGなど静的buildの場合にはスクロール位置を復元してくれますが、`getServerSideProps`利用時にはこのフラグを有効にしないとスクロール位置が復元されません。最近この辺りについて識者の方々から色々アドバイスももらい、自分では気付けないような部分の知見も多く得られたので、備忘録兼ねて`scrollRestoration`が何を解決しようとして、どう実装されているのか解説したいと思います。

## Next.jsのスクロール復元挙動

多くのMPA(Multi Page Application)では、ブラウザバック/フォワードを行った際にはスクロール位置はブラウザによって復元されます。こういった挙動が各ブラウザにいつからあるのかは不明ですが、スマホでのブラウジング(特にスワイプ)との相性を考えると自然な挙動に思えます。この辺りの使用についてはwhatwgでも明記されています。

https://html.spec.whatwg.org/multipage/browsing-the-web.html#persisted-user-state-restoration

> If entry's scroll restoration mode is "auto", then the user agent may use entry's scroll position data to restore the scroll positions of entry's document's restorable scrollable regions.

browser historyとして格納されるentry stateにはscroll position dataというものがあり、「scroll restoration modeが`auto`(`History.scrollRestoration = auto`と同義)ならscroll position dataをもとにスクロール位置を復元することができる」とされています。

Next.js始めSPAでは、これらの挙動がうまく動作しないことがあります。

### `getServerSideProps`なしのページの場合

`getServerSideProps`を含まない場合、ページのレンダリングは即座に行われます。ブラウザは`popstate`イベントで行われる処理の完了を待ってスクロール位置を復元していると思われるため、正常に位置は復元されます。

- demoの動画

### `getServerSideProps`ありのページの場合

- 非同期処理の場合、ブラウザ側での復元が先にきてしまう
- demoの動画

### SPAではこう言った問題起きがち

- こう言った問題を scroll amnesiaと呼ぶ
  - MPAだとブラウザがやってくれてるがSPAだと起きがち
- Vercelの社長の解説

### `experimental.scrollRestoration`

- このフラグによってgSSPもサポートできる
- 内部的にはgSSP完了後に頑張って位置を復元してる

## next.jsのscrollRestoration実装

### 12.2.0まで

- indexを内部にもってる
- 遷移時やpopstate時に更新してる

### 12.2.0以降

- 連番はよく壊れる
- PRのリンク

### 残ってる問題点

- リロード時に復元できない
- PRのリンク

## 余談:Next.jsで状態を復元する

- スクロールだけでなく、状態もよく失われれる
- recoil + recoil-sync + koichikさんが作ったライブラリでそれが補完できる
- recoil-sync-next
- 近々こっちも解説詳細に書こうと思う
