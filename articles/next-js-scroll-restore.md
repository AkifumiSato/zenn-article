_---
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

### getServerSideProps

`getServerSideProps`を含むページの場合、遷移時やブラウザバック時にNext.jsは`getServerSideProps`をサーバー側で実行しpropsを取得するようなfetchを実行します。そのため、これらを含む場合ブラウザバック時のレンダリングは非同期的になります。

以下はChromeで挙動を確認した際のgifです。

:::details chrome demo
![](/images/next-js-scroll-restore/default_gssp.gif)
:::

このデモでは、`getServerSideProps`に以下のようにsleep処理を入れてレスポンスに時間がかかるようにしています。

```ts
export const getServerSideProps = async () => {
  await sleep(1000)
  return {
    props: {
      text: 'hello, world',
    },
  }
}
```

another pageからブラウザバックする際に先にURLが変更され、少し時間を置いて画面が描画されている様子が伺えます。このように、`getServerSideProps`を含むページでは**スクロール位置がブラウザによって復元されません**。

### getStaticProps

`getStaticProps`を含む場合のブラウザバックでも、`getServerSideProps`同様にNext.jsの内部処理によって`getStaticProps`の戻り値を取得するようなfetchが実行されます。この際の大きな違いはfetchによって取得されるのが静的なjsonであるという点と、多くの場合fetch先からは304 Not Modifiedがレスポンスされる点です(CDNやNext.jsが動くサーバーを想定)。

ただし、`getStaticProps`は`getServerSideProps`同様レンダリングには非同期処理を伴いますが、筆者の環境では**Chromeではスクロール位置が復元され、Safariでは復元されませんでした**。

:::details chrome demo
![](/images/next-js-scroll-restore/default_gsp.gif)
:::

実装を確認した限り、Next.jsはデフォルトではスクロール位置を復元しないこと、そしてSafariでは復元されない(=ブラウザ差異がある)ことからもChromeのスクロール位置の復元はChromeの実装によるものだと考えられます。各ブラウザがどのタイミングで復元処理を実行するか、そもそもどう言った条件で実行するのかどうかについてはwhatwgの仕様的にはブラウザ側で自由に実装できそうなので、Chromeではかなり複雑な条件をもとに判定しているのかもしれません。

### Data fetchingを含まない場合

`getServerSideProps`や`getStaticProps`といったData fetching機能を含まないページの場合、動的なprops取得がないためにブラウザバック時のレンダリングにはfetchは含まれず、直ちにレンダリングすることが可能です。ただしこの際にもスクロール位置復元されるかどうかについてはブラウザによって差異があり、Chromeでは復元されますがSafariでは復元されませんでした。

:::details chrome demo
![](/images/next-js-scroll-restore/default_static.gif)
:::

## Scroll amnesia

こういったスクロール位置の喪失は**Scroll amnesia**と呼ばれます。SPAではScroll amnesiaであったり、状態の復元というのが軽視されがちな状況にあります。SPAの擬似遷移はシームレスな遷移や必要最低限のネットワークのラウンドトリップによる高速化が見込める一方で、スクロールや状態の復元の体験がMPAと異なることでUX的には一長一短という状態になってしまっているようにも感じます。

こういった状況を改善するためにも、whatwgでは[Navigation API](https://github.com/WICG/navigation-api/blob/main/README.md)という新たなブラウザAPIによってSPAの擬似遷移の体験改善が検討されています。

https://blog.jxck.io/entries/2022-04-22/navigation-api.html#%E3%83%95%E3%82%A9%E3%83%BC%E3%82%AB%E3%82%B9%E3%81%AE%E7%AE%A1%E7%90%86

## experimental.scrollRestorationの実装

Navigation APIはまだChromeでしか実装されておらず、実用の段階にはないたSPAでScroll amnesiaに対応するには別途実装が必要です。これを有効にするのが冒頭で述べた`experimental.scrollRestoration`フラグです。このフラグを有効にした際にNext.jsがどうやって位置を復元しているか見てみましょう。

### 処理の概要

大まかな処理の概要は以下です。

1. 履歴に対してkeyを作成
2. `next/link`で遷移時(`pushState`)時にkeyとスクロール位置をSession Storageに保存
3. ブラウザバック/フォワード時にkeyをもとにSession Storageからスクロール位置を取得
4. Next.jsのrender関数内でスクロール位置を復元

これらの処理について、詳細にみていきましょう。

#### 1. 履歴に対してkeyを作成

Next.jsには`next/link`や`useRouter`などで利用される`Router` classが存在します。主にこのclassはクライアントサイドで利用され、サーバー側では共通のinterfaceをもった`ServerRouter`classが利用されます。

https://github.com/vercel/next.js/blob/v12.2.0/packages/next/shared/lib/router/router.ts#L620

`Router`はクライアントサイドのルーティングや[pushState](https://developer.mozilla.org/ja/docs/Web/API/History/pushState)によるURL更新、`getServerSideProps`を実行するfetchの発行など、Next.jsのライフサイクルに対する責務を負っています。この`Router`で、新規の履歴に対してユニークなkeyが作成されます。

実は12.2.0まではkeyはユニークな文字列ではなく単純なindexでしたが、indexだと容易に履歴が破綻するためリロードを挟むと復元できなくなっていました。これを修正するためにユニークなkeyで管理するよう修正したプルリクをNext.jsに投げて、無事マージ・リリースされたので現在はユニークな文字列がkeyとなっています(Next.jsへ初PR&マージでした、やったー！)。

https://github.com/vercel/next.js/pull/36861

#### 2. keyとスクロール位置をSession Storageに保存

#### 3. ブラウザバック/フォワード時にkeyをもとにスクロール位置を取得

#### 4. Next.jsのrender関数内でスクロール位置を復元

### 残ってる問題点

- リロード時に復元できない
  - PRのリンク

## (予想)Navigation APIを利用した実装

## 余談:Next.jsで状態を復元する

- スクロールだけでなく、状態もよく失われれる
- Vercelの社長も大事な原則として上げてる
  - https://yosuke-furukawa.hatenablog.com/entry/2014/11/14/141415#5
- recoil + recoil-sync + koichikさんが作ったライブラリでそれが補完できる
- recoil-sync-next
- 近々こっちも解説詳細に書こうと思う_
