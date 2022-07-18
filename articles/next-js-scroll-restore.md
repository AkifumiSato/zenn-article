---
title: "Next.jsはどうやってスクロール位置を復元するのか"
emoji: "📜"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: true
---

[Next.js](https://nextjs.org/)にはexperimental(実験的機能)で`scrollRestoration`というフラグが存在します。

```js
// next.config.js
const nextConfig = {
  // ...
  experimental: {
    scrollRestoration: true,
  },
}

module.exports = nextConfig
```

デフォルトでもブラウザ側でスクロール位置を復元してくれることもありますが、Safariでは復元されなかったり、Chromeでも`getServerSideProps`利用時にはこのフラグを有効にしないとスクロール位置が復元されないなど不安定な状態です。最近この辺りについて識者の方々から色々ご教示いただき、自分では気付けないような部分の知見も多く得られたので、備忘録兼ねて`scrollRestoration`が何を解決しようとして、どう実装されているのか解説したいと思います。

## Next.jsのスクロール復元挙動

多くのMPA(Multi Page Application)では、ブラウザバック/フォワードを行った際にはスクロール位置はブラウザによって復元されます。こういった挙動が各ブラウザにいつからあるのかはわかりませんが、スマホでのブラウジング(特にスワイプ)との相性を考えると自然な挙動に思えます。この辺りの仕様についてはwhatwgでも明記されています。

https://html.spec.whatwg.org/multipage/browsing-the-web.html#persisted-user-state-restoration

> If entry's scroll restoration mode is "auto", then the user agent may use entry's scroll position data to restore the scroll positions of entry's document's restorable scrollable regions.

ブラウザのhistory entryに格納されるstateにはscroll position dataというものがあり、「scroll restoration modeが`auto`ならscroll position dataをもとにuser agentはスクロール位置を復元することができる」とされています。scroll restoration modeは[history.scrollRestoration](https://developer.mozilla.org/ja/docs/Web/API/History/scrollRestoration) に`auto`(初期値)か`manual`を代入することで設定できます。

Next.jsはデフォルトでは`experimental.scrollRestoration = false`となっており、この場合の`history.scrollRestoration`の値は`'auto'`です。SPAでは、`history.scrollRestoration = 'auto'`によるブラウザ側の復元処理がうまく動作しないことがあります。

Next.jsにおける`history.scrollRestoration = 'auto'`の挙動はSSGなページかSSRなページかによって変わってきます。

### getServerSideProps

`getServerSideProps`を含むページの場合、遷移時やブラウザバック時にNext.jsは`getServerSideProps`をサーバー側で実行しpropsなどの戻り値を取得するようなfetchを実行します。そのため、これらを含む場合ブラウザバック時のレンダリングは非同期的になります。

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

another pageからブラウザバックする際に先にURLが変更され、少し時間を置いて画面が描画され、スクロール位置は先頭に戻っているのが確認できます。このように、`getServerSideProps`を含むページでは**スクロール位置がブラウザによって復元されません**。

### getStaticProps

`getStaticProps`を含む場合のブラウザバックでも、`getServerSideProps`同様にNext.jsの内部処理によって`getStaticProps`のpropsなどの戻り値を取得するようなfetchが実行されます。この際の大きな違いはfetchによって取得されるのが静的なjsonであるという点と、多くの場合(CDNやNext.jsが動くサーバーを想定)fetch先からは304 Not Modifiedがレスポンスされる点です。

ただし、`getStaticProps`は`getServerSideProps`同様レンダリングには非同期処理を伴いますが、筆者の環境では**Chromeではスクロール位置が復元され、Safariでは復元されませんでした**。

:::details chrome demo
![](/images/next-js-scroll-restore/default_gsp.gif)
:::

実装を確認した限り、Next.jsはデフォルトではスクロール位置を復元しないこと、そしてSafariでは復元されない(=ブラウザ差異がある)ことからもChromeのスクロール位置の復元はChromeの実装によるものだと考えられます。各ブラウザがどのタイミングで復元処理を実行するかは、whatwgの仕様の「スクロール位置を復元することができる」というニュアンスからもブラウザ側で自由に実装できそうなので、Chromeでは「一定時間内のfetchが304の場合」などかなり複雑な条件をもとに判定しているのかもしれません。

### Data fetchingを含まない場合

`getServerSideProps`や`getStaticProps`といったData fetching機能を含まないページの場合、動的なprops取得がないためにブラウザバック時のレンダリングにはfetchは含まれず、直ちにレンダリングすることが可能です。ただしこの場合も`getStaticProps`と同様、**Chromeではスクロール位置が復元され、Safariでは復元されませんでした**。

:::details chrome demo
![](/images/next-js-scroll-restore/default_static.gif)
:::

## Scroll of amnesia

こういったスクロール位置の喪失は**Scroll of amnesia**と呼ばれます。SPAではScroll of amnesiaであったり、状態の復元というのが軽視されがちな状況にあります。SPAの擬似遷移はシームレスな遷移や必要最低限のネットワークのラウンドトリップによるページ描画の高速化が見込める一方で、スクロールや状態の復元を軽視してしまうとブラウザバック/フォワードのUXが悪化してしまいます。

こういった状況を改善するためにも、whatwgでは[Navigation API](https://github.com/WICG/navigation-api/blob/main/README.md)という新たなブラウザAPIによってSPAの擬似遷移の体験改善が検討されています。

https://blog.jxck.io/entries/2022-04-22/navigation-api.html#%E3%83%95%E3%82%A9%E3%83%BC%E3%82%AB%E3%82%B9%E3%81%AE%E7%AE%A1%E7%90%86

## experimental.scrollRestorationの実装

Navigation APIはまだChromeでしか実装されておらず、実用の段階にはないためSPAでScroll of amnesiaに対応するには別途実装が必要です。これを有効にするのが冒頭で述べた`experimental.scrollRestoration`フラグです。このフラグを有効にした際にNext.jsがどうやってスクロール位置を復元しているか見てみましょう。

### 処理の概要

大まかな処理の概要は以下です。

1. 現在の履歴(history)に対してkeyを作成
2. 遷移前にkeyとスクロール位置をSession Storageに保存
3. ブラウザバック/フォワード時にkeyをもとにSession Storageからスクロール位置を取得
4. Next.jsのrender関数内でスクロール位置を復元

これらの処理について、詳細にみていきましょう。

#### 1. 現在の履歴に対してkeyを作成

Next.jsには`next/link`や`useRouter`などで利用される`Router` classが存在します。主にこのclassはクライアントサイドで利用され、サーバー側では共通のinterfaceをもった`ServerRouter`classが利用されます。

https://github.com/vercel/next.js/blob/v12.2.0/packages/next/shared/lib/router/router.ts#L620

`Router`はクライアントサイドのルーティングや[pushState](https://developer.mozilla.org/ja/docs/Web/API/History/pushState)によるURL更新、`getServerSideProps`を実行するfetchの発行など、Next.jsのライフサイクルに対する責務を負っています。この`Router`内で、新規の履歴に対してユニークなkeyが作成されます。

実は12.2.0まではkeyはユニークな文字列ではなく単純なindexでしたが、indexだと容易に履歴が破綻するためリロードを挟むなどすると復元できなくなっていました。これを修正するためにユニークなkeyで管理するよう修正したプルリクをNext.jsに投げて、無事マージ・リリースされたので現在はユニークな文字列がkeyとなっています(識者の方にアドバイスしてもらいながらNext.jsへ初PR&マージでした、多謝🙇‍♂️)。

https://github.com/vercel/next.js/pull/36861

#### 2. 遷移前にkeyとスクロール位置をSession Storageに保存

`next/link`やブラウザバック/フォワードでの**遷移前**にスクロール位置をSession Storageに`__next_scroll_${key}`といキー名称で保存します。

_popstate時_
https://github.com/vercel/next.js/blob/v12.2.0/packages/next/shared/lib/router/router.ts#L863-L866

_遷移時_
https://github.com/vercel/next.js/blob/v12.2.0/packages/next/shared/lib/router/router.ts#L937-L940

#### 3. ブラウザバック/フォワード時にkeyをもとにスクロール位置を取得

ブラウザバック/フォワードで**遷移後**のkeyをもとに、Session Storageから`__next_scroll_${key}`といキー名称で`getItem`して、あればその値をスクロール位置情報として取得します。

https://github.com/vercel/next.js/blob/v12.2.0/packages/next/shared/lib/router/router.ts#L870-L875

#### 4. Next.jsのrender関数内でスクロール位置を復元

取得したスクロール位置の復元情報を、クライアントサイドのrender処理(`doRender`)の最後で`window.scrollTo`に渡してスクロール位置を復元します。

https://github.com/vercel/next.js/blob/v12.2.0/packages/next/client/index.tsx#L980-L982

### 残ってる問題点

以上が`experimental.scrollRestoration`有効時の復元処理になります。ただし、この復元処理にはまだ課題があります。リロード時に`Router`の`key`がリセットされてしまうため、リロード時のみスクロール位置が復元できないのです。

これについても修正プルリクを作成しています。

https://github.com/vercel/next.js/pull/37127

現在Vercel内で実装が進んでると思われる[Nested Layout](https://nextjs.org/blog/layouts-rfc)によっておそらく`Router`も大きく変更されるでしょうから、その前にマージしてもらえたら嬉しいのですが、忙しいのか競合するからかなかなかレビューしてもらえません。根気強く待とうかと思います。

## 将来的な実装: Navigation APIの利用

将来的にNavigation APIが広くサポートされNext.jsでも利用されることになった場合には、上述の処理は不要になります。Navigation APIでは`navigate`イベント時に**遷移を定義できるようになります**。遷移を定義できるというのは、ブラウザ側に「遷移の開始〜終了」をPromiseで渡すAPIを用意することで、ブラウザ側がMPAでしかできなかったローダーの表示やブラウザによるスクロール位置の復元を行えるようになることを意味します。

```js
navigation.addEventListener("navigate", e => {
  e.transitionWhile((async () => {
    const data = await fetchData()
    await render(data)
  })())
})
```

https://twitter.com/domenic/status/1471604621470846979

必要であれば、`e.restoreScroll()`というメソッドを呼び出せば`transitionWhile`に渡すPromiseの任意のタイミングでスクロール位置を復元できるので、履歴のkey管理やSession Storageへのスクロール位置情報の格納は不要になります。

```js
navigation.addEventListener("navigate", e => {
  e.transitionWhile((async () => {
    const data = await fetchData()
    await render(data)
    e.restoreScroll() // スクロール位置を復元
    // renderに関係ない処理
    await sendMeasurement()
    await sendReporting()
  })())
})
```

詳しくは[前述のjxckさんの記事](https://blog.jxck.io/entries/2022-04-22/navigation-api.html#%E3%83%95%E3%82%A9%E3%83%BC%E3%82%AB%E3%82%B9%E3%81%AE%E7%AE%A1%E7%90%86)を参照ください。

## まとめ

ブラウザの復元処理とNext.jsの挙動、`experimental.scrollRestoration = true`でどう挙動が変わりどう実装されているのかみてきました。jxckさんの記事にもありますが、スクロール位置の復元の体験はSPAでは軽視されがちです。これらの複雑な実装やUXの更なる改善が期待できるNavigation APIが早くChrome以外も対応してくれることを願っています。

同様にSPAで軽視されがちなものとして、UI状態の復元が挙げられます。こちらについては本稿ではあまり触れませんでしたが、以前書いた[スコープとライフタイムで考えるReact State再考](https://zenn.dev/akfm/articles/react-state-scope)という記事で詳しく触れています。

https://zenn.dev/akfm/articles/react-state-scope

この記事の後[recoil-sync](https://recoiljs.org/docs/recoil-sync/introduction/)がリリースされ、recoil-syncとNext.jsを連携する[recoil-sync-next](https://github.com/recruit-tech/recoil-sync-next)というライブラリ開発に微力ながら携わらせていただきました。

https://twitter.com/koichik/status/1547866613944201218

近々こちらについてもできれば記事にしたいなと思います。
