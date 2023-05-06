---
title: "Next.js App Routerの遷移はどう実現しているのか"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

Next.jsのv13.4が発表され、[App RouterがStable](https://nextjs.org/blog/next-13-4#nextjs-app-router)になりました。App Routerは発表以来、猛烈なスピードで実装が進んでおり、最近も[Server Action](https://nextjs.org/blog/next-13-4#server-actions-alpha)や[Parallel Routes](https://nextjs.org/blog/next-13-3#parallel-routes-and-interception)などの新機能が次々と発表されています。

当然ながらこれらの話題はフレームワーク利用者目線の話題が多いのですが、本稿はApp Routerがどう実装されているのか、筆者の興味のままに遷移処理周りを中心に調査した際のまとめ記事になります。あまり話題にはなってませんが知っておくと役に立つこともあるかと思うので、参考にうなれば幸いです。

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

`pages`の場合、**静的ファイル**は積極的にprefetchされます。ここで言う静的ファイルとは、JSファイルや画像を指すわけですが、SSGされたページはJSファイルに含まれるので、概念的にはSSGページもprefetchされると考えて良いでしょう。

![](/images/next-app-router-navigation/pages-prefetch.png)

一方でSSRページの場合、`getServerSideProps`の結果だけはリンクが実際に押下された時に、JSONで取得されます。具体的には`http://localhost:3000/_next/data/Czj63Y47_EGJEXlNGwfI6/pages/example_dynamic.json`のようなURLでfetchしてJSONを取得します。そのため、**押下直後のリクエスト発火からレスポンスを受け取るまでの間は、直前の画面が表示されることになります**。

以下で言うと、1~4まではずっと直前の画面が表示されていることになります。

1. あらかじめJSファイルなどはprefetchされる
1. SSRページに遷移するリンク押下
1. `getServerSideProps`の結果JSONをfetch開始
1. ↑のfetchレスポンスを受け取る
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

App Routerはこのような動的処理を含むページでも**積極的にprefetchします**。

![](/images/next-app-router-navigation/app-prefetch.png)

App Routerによるprefetch時のURLはページとまったく同じ`http://localhost:3000/app/example_dynamic`という形ですが、`RSC: 1`というhttpヘッダーを含み、このヘッダーがあるときは文字通りReact Server Components（略して`RSC`）としてレスポンスされます。ブラウザのURLアドレスバーで上記URLを入力すると、当然このヘッダーがついていないのでhtmlとしてレスポンスされます。

このように、App Routerは積極的にprefetchを行うことで、従来の`pages`とは違い直前の画面が表示されて待たされると言うことがほぼなく、**即座に遷移が発生します**。もちろんprefetchはOFFにすることが可能で、OFFの際には`pages`同様fetch完了までは直前の画面が表示されますが、デフォルト挙動が変更されたのは大きな方針変更と言えるかと思います。

個人的見解ですが、App Routerは積極的なCache戦略を展開しているので、積極的なprefetchによってサーバー負荷を高める懸念をCache戦略を持ってカバーしているというスタンスなのではないかと思います。

App Routerのprefetchはさらに細く仕様が存在しますが、これらは後述します。

### `pages`と`app`の境界を越える際の挙動

- これらの境界を越える際の挙動
  - pagesとapp routerは現在、併用が可能
  - しかし、これらは仕様も実装も大きく異なる
  - これは、これらの境界を越えることを判定し、MPA遷移（JS制御による擬似遷移ではなく、ブラウザのデフォルトの遷移挙動）になる
    - Server Componentのレンダリング結果は、Flight Protocolというデータ形式で返却される
    - Flightでない時にはapp間の遷移でないと判定され、MPA遷移

## 構成

- app routerの遷移哲学
  - app routerでは、積極的なprefetchによりCore Web Vitalsに新たに追加された指標であるINP（Interaction To Next Paint）が改善された遷移を提供
  - これにより、ユーザーは画面がフリーズして見えることが少なく
    - パターンが多く仕様を全部把握するのは困難だが、筆者が動作確認した限りにおいてはなかった
  - 積極的なprefetchをおこなっているので、サーバー負荷的にはfriendlyではない
- app routerの遷移実装
  - この遷移挙動をapp routerはどうやって実現しているのか、実装を追ってみる
  - 実はwindowの`nd.router`にもapp routerのインスタンスがいるので、debugしたい人はそこから辿ってみるのもいいかもしれません
  - 内部的にはReduxを使って状態管理を行なっている
    - RTKなどは使わず、ほぼ素のRedux。副作用はコンポーネント側責務としているっぽい
  - prefetch,navigateアクションなどがある
  - Linkコンポーネントはvisible,hover,touchStartでprefetchアクションが発火する
    - prefetch処理自体は、Redux Stateで`Mpa<Url, Promise<Data>>`の形で保持される
      - PromiseをState自体に含めてしまうやり方は初めて見たのでちょっと驚いた
      - middlewareとかでやればいいのに、なぜこんなやり方をしてるのかは謎
        - reducerの純粋性を失ってまでReduxを使う意味があったのだろうか...？
      - prefetch自体は普通のGETリクエストで、`RSC: 1`によってServer Componentを判断してる（なければhtml）
  - Link押下時には、prefetchした結果を元に遷移を決定する
    - prefetchから結果を読み取る
      - RSCのレスポンス形式はFlight ProtocolやReact Flightなどと呼ばれる、少しJSONに似た独自のデータ形式
      - prefetch結果がFlightなら、app router遷移
    - prefetchデータがstring=pagesへの遷移だったり、遷移先が外部URLならMPA遷移
      - MPA遷移はRedux Stateの`pushRef.mpaNavigatio`がtrueに変更され、HistoryUpdaterコンポーネント内で読み取られ`location.assign`が呼ばれる
    - prefetch失敗時はMPA遷移となる
  - App Routerのnavigation発生後
    - コンポーネントごとReduxにcacheを持っておき、それをレンダリングしてる
      - https://github.com/vercel/next.js/blob/285e77541f/packages/next/src/client/components/app-router.tsx#L372
    - `walkTreeWithFlightRouterState`によって再起的にLayoutを解決してる
    - 解決されたLayoutを元に、Client側ではキーを元に描画するコンポーネントをCacheから引いて決定してる
      - todo: 説明が不足しそうなので文章構成よく考える
- Intercept routing
  - Interception時はprefetchの結果が異なる
    - `rewrites`の1つとしてルーティングが作成される
      - haederの`Next-Url`が正規表現と一致した時のみ、intercept routingにマッチする
      - https://github.com/vercel/next.js/blob/285e77541f/packages/next/src/lib/generate-interception-routes-rewrites.ts#L66-L76
    - `rewrites`のルールなどは`routes-manifest.json`に吐き出される
      - このjsonの内容を元にルーティングが決定されるっぽい
        - https://github.com/vercel/next.js/blob/285e77541f/packages/next/src/server/next-server.ts#L1198
      - TODO: このJSONの中身のサンプルを作成する

