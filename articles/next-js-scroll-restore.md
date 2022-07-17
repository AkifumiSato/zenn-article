---
title: "Next.jsはどうやってスクロール位置を復元するのか"
emoji: "📜"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

[Next.js](https://nextjs.org/)にはexperimental(実験的機能)で`scrollRestoration`というフラグが存在します。デフォルトでもブラウザ側でスクロール位置を復元してくれることもありますが、Safariでは復元されなかったり、Chromeでも`getServerSideProps`利用時にはこのフラグを有効にしないとスクロール位置が復元されないなど不安定な状態です。最近この辺りについて識者の方々から色々ご教示いただき、自分では気付けないような部分の知見も多く得られたので、備忘録兼ねて`scrollRestoration`が何を解決しようとして、どう実装されているのか解説したいと思います。

## Next.jsのスクロール復元挙動

多くのMPA(Multi Page Application)では、ブラウザバック/フォワードを行った際にはスクロール位置はブラウザによって復元されます。こういった挙動が各ブラウザにいつからあるのかは把握していませんが、スマホでのブラウジング(特にスワイプ)との相性を考えると自然な挙動に思えます。この辺りの使用についてはwhatwgでも明記されています。

https://html.spec.whatwg.org/multipage/browsing-the-web.html#persisted-user-state-restoration

> If entry's scroll restoration mode is "auto", then the user agent may use entry's scroll position data to restore the scroll positions of entry's document's restorable scrollable regions.

ブラウザのhistory entryに格納されるstateにはscroll position dataというものがあり、「scroll restoration modeが`auto`ならscroll position dataをもとにスクロール位置を復元することができる」とされています。scroll restoration modeは[history.scrollRestoration](https://developer.mozilla.org/ja/docs/Web/API/History/scrollRestoration) に`auto`(初期値)か`manual`を代入することで設定できます。

Next.jsはデフォルトでは`experimental.scrollRestoration = false`となっており、この場合の`history.scrollRestoration`の値は`auto`です。Next.js始めSPAでは、`history.scrollRestoration = 'auto'`によるブラウザ側の復元処理がうまく動作しないことがあります。

### `getServerSideProps`

`getStaticProps`や`getServerSideProps`を含む場合、遷移時やブラウザバック時にNext.jsは該当のprops取得処理に相当するfetchを実行します。そのため、これらを含む場合にはブラウザバック時のレンダリングは非同期的になります。

- 非同期処理の場合、ブラウザ側での復元が先にきてしまう
- demoの動画

### `getStaticProps`

`getStaticProps`を含む場合のブラウザバックでは、Next.jsの内部処理によって`getStaticProps`の戻り値を取得するようなfetchが実行されるため、画面の描画は非同期的になります。ただし、この場合fetch先からは304 Not Modifiedがレスポンスされ、`getStaticProps`を含まない時と同様にChromeではスクロール位置が復元されます。

:::details chrome demo
![](/images/next-js-scroll-restore/default_gssp_demo.gif)
:::

### Data fetchingを含まない場合

`getServerSideProps`や`getStaticProps`といったData fetching機能を含まないページの場合、動的なprops取得がないためにブラウザバック時のレンダリングにはfetchは含まれず、直ちにレンダリングすることが可能です。ただしこの時もスクロール位置復元されるかどうかについてはブラウザによって差異があり、Chromeでは復元されますがSafariでは復元されませんでした。

:::details chrome demo
![](/images/next-js-scroll-restore/default_static.gif)
:::

### まとめ

- ブラウザとData fetchで対応表を作成

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

## (予想)Navigation APIを利用した実装

## 余談:Next.jsで状態を復元する

- スクロールだけでなく、状態もよく失われれる
- recoil + recoil-sync + koichikさんが作ったライブラリでそれが補完できる
- recoil-sync-next
- 近々こっちも解説詳細に書こうと思う
