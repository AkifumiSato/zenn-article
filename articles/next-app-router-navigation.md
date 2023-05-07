---
title: "Next.js App Routerの遷移はどう実現しているのか"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

Next.jsのv13.4が発表され、[App RouterがStable](https://nextjs.org/blog/next-13-4#nextjs-app-router)になりました。App Routerは発表以来、猛烈なスピードで実装が進んでおり、最近も[Server Action](https://nextjs.org/blog/next-13-4#server-actions-alpha)や[Parallel Routes](https://nextjs.org/blog/next-13-3#parallel-routes-and-interception)などの新機能が次々と発表されています。

当然ながらこれらの話題はフレームワーク利用者目線の話題が多いのですが、本稿はApp Routerがどう実装されているのか、筆者の興味のままに遷移処理周りを中心に調査した際のまとめ記事になります。あまり話題にはなってませんが知っておくと役に立つものもあるかと思うので、参考になれば幸いです。

:::message
- 細かい仕様や内部実装の話がほとんどで、機能の説明などは省略しているのでそちらは[公式ドキュメント](https://nextjs.org/docs)や他の記事をご参照ください。
- 「遷移処理周り」といっても仕様や実装は膨大なので、筆者の興味の向くままに大枠を調査したものです。
- 実装は当然ながらアップデートされるため、記事の内容が最新にそぐわない可能性があります。執筆時に見てたコミットは[9028a16](https://github.com/vercel/next.js/tree/9028a169acb04c208844582866c7317dfc336580)です。
:::

## Next.jsの遷移とprefetch挙動

Next.jsの遷移を理解するには、まずprefetch挙動について知る必要があります。今回は調査用のデモとして、App Router（以降`app`）と`pages`それぞれで、同じようなページをいくつか用意しました。

![](/images/next-app-router-navigation/demo-app-top.png)
*`app`で実装したページ*

![](/images/next-app-router-navigation/demo-pages-top.png)
*`pages`で実装したページ*

これらで従来の`pages`と`app`の挙動を比較していきたいと思います。

### `pages`の挙動

`pages`の場合、**静的ファイルは積極的にprefetch**されます。

![](/images/next-app-router-navigation/pages-prefetch.png)

ここで言う静的ファイルとは、JSファイルや画像を指すわけですが、SSGされたページはJSファイルに含まれるので、概念的にはSSGページもprefetchに含まれると考えて良いでしょう。

一方でSSRページの場合、`getServerSideProps`の結果だけはリンクが実際に押下された時に、JSONで取得されます。具体的には`http://localhost:3000/_next/data/Czj63Y47_EGJEXlNGwfI6/pages/example_dynamic.json`のようなURLでfetchしてJSONを取得します。そのため、**押下直後のリクエスト発火からレスポンスを受け取るまでの間は、直前の画面が表示されることになります**。

以下で言うと、1~4まではずっと直前の画面が表示されていることになります。

1. あらかじめJSファイルなどはprefetchされる
1. SSRページに遷移するリンク押下
1. `getServerSideProps`の結果をfetch開始
1. ↑のレスポンスをJSONで受け取る
1. ページ遷移

### `app`の挙動

一方でApp Routerには`getServerSideProps`はなく、Server Componentsのレンダリング時にデータ取得が行われます。

```tsx
export default async function Products() {
  const res = await fetch("https://dummyjson.com/products");
  const data = await res.json();

  return (
    ...
  );
}
```

App Routerはこのような**動的処理を含むページでも積極的にprefetch**します。

![](/images/next-app-router-navigation/app-prefetch.png)

App Routerによるprefetch時のURLはページとまったく同じ`http://localhost:3000/app/example_dynamic`という形ですが、`RSC: 1`というhttpヘッダーを含み、このヘッダーがあるときは文字通りReact Server Components（略して`RSC`）としてレスポンスされます。ブラウザのURLアドレスバーで上記URLを入力すると、当然このヘッダーがついていないのでhtmlとしてレスポンスされます。

このように、App Routerは積極的にprefetchを行うことで、従来の`pages`とは違い直前の画面が表示されて待たされると言うことがほぼなく、**即座に遷移が発生します**。もちろんprefetchはOFFにすることが可能で、OFFの際には`pages`同様fetch完了までは直前の画面が表示されますが、デフォルト挙動が変更されたのは大きな方針変更と言えるかと思います。

App Routerのprefetchはさらに細く仕様が存在しますが、これらは後述します。

### `pages`と`app`の境界を越える際の挙動

ここまで`pages`と`app`で分けて見てきましたが、仕様上これらは共存可能です。仕様も実装も大きく異なるRouterを持ったこれらのページを、どうやってNext.jsは共存させているのでしょう？

これを実現するのは単純な仕組みで、`pages`と`app`の境界を越える時にはSPA遷移ではなくMPA（=Multi Page Application）遷移、つまりJS制御による擬似遷移ではなくブラウザのデフォルト挙動の遷移を行うことで実現しています。

`pages`から`app`への段階的リリースを伴うマイグレーションを検討している方は、移行段階においてSPA遷移が一部失われることを認識しておくと良いかと思います。

## App Routerは遷移体験の改善を目論む

積極的なprefetchによって実現されるのは高いパフォーマンスです。具体的には新たにCore Web Vitalsに追加されたINP（**Interaction To Next Paint**）が改善されたSPA遷移を提供することを目指していると考えられます。前述の通り、従来の`pages`では画面押下直後は画面が応答していないように見えてしまうことから、INP的に優れた体験とは言えませんでした。App Routerでは積極的なprefetchによって、この問題を解決しようとしていると考えられます。

一方でこの積極的なprefetchは、BFFの上側にあるであろうAPIサーバーやデータベース負荷を高めてしまうことにつながるであろうことは明白です。個人的見解を含みますが、App Routerはこの負荷的な懸念を、Cache戦略を持ってカバーしているというスタンスなのではないかと思います。fetch単位・page単位のrevalidateを使いこなすことで、ユーザーにとってのパフォーマンスだけでなく負荷軽減も見込めます。また、Layoutという概念の登場により、前のページと比較して必要なLayout部分のみレンダリングすることも、負荷軽減が見込めます。

## App Routerの遷移実装

ここまでApp Routerの遷移やprefetchの仕様について見てきましたが、ここからはこれらの実装を追ってみたいと思います。

### App Routerの状態管理

App Routerは内部的に状態管理を`useReducer`を使って作成した、Reduxもどきで管理しています。ここであえてReduxもどきと表現したのは、Redux Devtoolsと連携しているためです。Redux Devtoolsを入れていると、App Routerの状態管理がRedux Devtoolsによって可視化されます。

ただし、後述しますがPromiseやReactElementをStateに含んでいるので、参考程度に見ることをお勧めします（筆者は最初これをあてにしすぎて、だいぶ混乱しました）。

### `prefetch`アクション

`next/link`から提供されるLinkコンポーネントはvisible、hover、touchStartイベント時に`router.prefetch`を呼び出します。

https://github.com/vercel/next.js/blob/9028a169ac/packages/next/src/client/link.tsx#L161

`router.prefetch`は外部URLやBot判定をしたのち、prefetchアクションを発行します。

https://github.com/vercel/next.js/blob/9028a169ac/packages/next/src/client/components/app-router.tsx#L269-L273

prefetchアクションのreducerでは、fetchを発行しますが**意図的にawaitしておらず**、Stateにそのまま含めています。具体的には、Stateの`prefetchCache`に`Map<string, PrefetchCacheEntry>`で保持します。

https://github.com/vercel/next.js/blob/9028a169ac/packages/next/src/client/components/router-reducer/reducers/prefetch-reducer.ts#L53-L62

このようにApp Routerのprefetchは、State上でPromiseごと保持しており、遷移時などでこのprefetchしたRSCを利用する時には、Promiseから取り出す形で実装されています。

### React Flight

prefetchリクエスト自体は前述の通り、ページURLに`RSC: 1`を含めたGETリクエストです。`RSC: 1`がなければhtmlを返します。RSCの場合は**Flight**や**React Flight**と呼ばれるRSC独自のデータフォーマットがレスポンスボディで返されます。

```
1:HL["/_next/static/css/5cc6c563bf8ab1da.css",{"as":"style"}]
0:[[["",{"children":["app",{"children":["example_static",{"children":["__PAGE__",{}]}]}]},"$undefined","$undefined",true],"$L2",[[["$","link","0",{"rel":"stylesheet","href":"/_next/static/css/5cc6c563bf8ab1da.css","precedence":"next.js"}]],["$L3",null]]]]
4:I{"id":"7846","chunks":["272:static/chunks/webpack-6365542cc30a6aab.js","769:static/chunks/8e422d1d-436056157c89b00f.js","365:static/chunks/365-6e63437f7129d097.js"],"name":"","async":false}
5:I{"id":"6650","chunks":["272:static/chunks/webpack-6365542cc30a6aab.js","769:static/chunks/8e422d1d-436056157c89b00f.js","365:static/chunks/365-6e63437f7129d097.js"],"name":"","async":false}
6:I{"id":"7371","chunks":["371:static/chunks/371-975f2f5092fc69c3.js","980:static/chunks/app/app/example_dynamic/page-2bccbb99522030af.js"],"name":"","async":false}
2:[["$","html",null,{"lang":"en","children":["$","body",null,{"children":["$","div",null,{"className":"flex min-h-screen flex-col items-center justify-between p-24","children":["$","div",null,{"className":"w-full max-w-5xl","children":["$","$L4",null,{"parallelRouterKey":"children","segmentPath":["children"],"error":"$undefined","errorStyles":"$undefined","loading":"$undefined","loadingStyles":"$undefined","hasLoading":false,"template":["$","$L5",null,{}],"templateStyles":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","asNotFound":false,"childProp":{"current":["$","$L4",null,{"parallelRouterKey":"children","segmentPath":["children","app","children"],"error":"$undefined","errorStyles":"$undefined","loading":"$undefined","loadingStyles":"$undefined","hasLoading":false,"template":["$","$L5",null,{}],"templateStyles":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","asNotFound":false,"childProp":{"current":["$","$L4",null,{"parallelRouterKey":"children","segmentPath":["children","app","children","example_static","children"],"error":"$undefined","errorStyles":"$undefined","loading":"$undefined","loadingStyles":"$undefined","hasLoading":false,"template":["$","$L5",null,{}],"templateStyles":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","asNotFound":false,"childProp":{"current":[["$","main",null,{"children":[["$","h1",null,{"className":"mb-4 text-3xl font-extrabold text-gray-900 dark:text-white md:text-5xl lg:text-6xl","children":[[["$","span",null,{"className":"text-transparent bg-clip-text bg-gradient-to-r to-emerald-600 from-sky-400","children":"`app`"}],"Â "],"example_static"]}],["$","p",null,{"className":"text-lg font-normal text-gray-500 lg:text-xl dark:text-gray-400","children":"This is an example page."}],["$","div",null,{"className":"mt-10","children":[["$","h2",null,{"className":"mb-4 text-xl font-extrabold text-gray-900 dark:text-white md:text-4xl lg:text-4xl","children":"Links"}],["$","ul",null,{"className":"list-decimal pl-5","children":[["$","li",null,{"children":["$","$L6",null,{"href":"/app","className":"underline","children":"/app"}]}],["$","li",null,{"children":["$","$L6",null,{"href":"/app/example_static","className":"underline","children":"/app/example_static"}]}],["$","li",null,{"children":["$","$L6",null,{"href":"/app/example_dynamic","className":"underline","children":"/app/example_dynamic"}]}],["$","li",null,{"children":["$","$L6",null,{"href":"/pages","className":"underline","children":"/pages"}]}],["$","li",null,{"children":["$","$L6",null,{"href":"/pages/example_static","className":"underline","children":"/pages/example_static"}]}],["$","li",null,{"children":["$","$L6",null,{"href":"/pages/example_dynamic","className":"underline","children":"/pages/example_dynamic"}]}]]}]]}]]}],null],"segment":"__PAGE__"},"styles":[]}],"segment":"example_static"},"styles":[]}],"segment":"app"},"styles":[]}]}]}]}]}],null]
3:[[["$","meta",null,{"charSet":"utf-8"}],["$","title",null,{"children":"Create Next App"}],["$","meta",null,{"name":"description","content":"Generated by create next app"}],null,null,null,null,null,null,null,null,["$","meta",null,{"name":"viewport","content":"width=device-width, initial-scale=1"}],null,null,null,null,null,null,null,null,null,null,[]],[null,null,null,null],null,null,[null,null,null,null,null],null,null,null,null,[null,[["$","link",null,{"rel":"icon","href":"/favicon.ico","type":"image/x-icon","sizes":"any"}]],[],null]]
```

Streamingを意識した仕様なのでしょうが、1行ずつ読むようなフォーマットになっています。先頭部分のみ省けば、JSON配列っぽく読めると思います。なんとなくコンポーネント情報やprops、childrenの情報などが見て取れます。おそらく`$`はコンポーネントを指しているように思うのですが、正確な仕様は不明です。

Flightの仕様とかを探してみたのですが、筆者には見つけることができませんでした。大抵は`react-server-dom-webpack`によってFlightのレスポンスを実現しているようですが、ここやreactのRFCにもFlightの仕様書などはなさそうでした。

https://github.com/facebook/react/tree/main/packages/react-server-dom-webpack

VercelにはReactコアチームのメンバーが多数いるので、この辺の仕様書はコアチーム内部に閉じてるのかもしれません。筆者が知らないだけで公開されてたらすいません、ご教示ください。

### `navigage`アクションと遷移判定

さて、Link押下時には、`navigate`アクションが発火します。`navigate`はいくつかのStateを更新して、遷移を指示するフラグやtreeを算出します。

具体的にはまず、prefetchした結果を元に遷移方法を決定します。prefetch結果がFlightでない場合、`pages`配下のページと判定し、MPA遷移となります。他にも外部URLの場合やprefetch失敗時にMPA遷移となります。これはStateの`pushRef.mpaNavigatio`がtrueに変更され、`Router`コンポーネント内で読み取られて`true`なら`location.assign`が呼ばれることで実現しています。

https://github.com/vercel/next.js/blob/9028a169ac/packages/next/src/client/components/app-router.tsx#L354-L357

`app`内での遷移の場合、Stateの`tree`が更新され、`InnerLayoutRouter`に渡されます。

https://github.com/vercel/next.js/blob/9028a169ac/packages/next/src/client/components/layout-router.tsx#L582-L593

この`InnerLayoutRouter`では`tree`に基づき順番に`subtree`が解決されていきます。

https://github.com/vercel/next.js/blob/9028a169ac/packages/next/src/client/components/layout-router.tsx#L429-L443

## 構成

- Intercept routing
  - Interception時はprefetchの結果が異なる
    - `rewrites`の1つとしてルーティングが作成される
      - haederの`Next-Url`が正規表現と一致した時のみ、intercept routingにマッチする
      - https://github.com/vercel/next.js/blob/285e77541f/packages/next/src/lib/generate-interception-routes-rewrites.ts#L66-L76
    - `rewrites`のルールなどは`routes-manifest.json`に吐き出される
      - このjsonの内容を元にルーティングが決定されるっぽい
        - https://github.com/vercel/next.js/blob/285e77541f/packages/next/src/server/next-server.ts#L1198
      - TODO: このJSONの中身のサンプルを作成する

