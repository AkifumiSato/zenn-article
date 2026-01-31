---
title: "Cache Componentsの世界観"
---

## 要約

Cache Componentsは[PPR↗︎](https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering)とCacheの改善^[参考記事: [Next.js Cache回想↗︎](https://zenn.dev/akfm/articles/nextjs-caching-history)]により、**デフォルトで高いパフォーマンス**と**優れた開発者体験**の両立を目指したものです。

Cache Componentsにおいてデータフェッチなど動的な処理を扱う際に、開発者は`<Suspense>`か`"use cache"`かどちらかを選択する必要があります。

## 背景

:::message
より詳細な経緯は筆者の過去記事[Next.js Cache回想↗︎](<https://zenn.dev/akfm/articles/nextjs-caching-history#%E6%94%B9%E5%96%841%3A-fetch()%E3%81%AE%E3%83%87%E3%83%95%E3%82%A9%E3%83%AB%E3%83%88cache%E5%BB%83%E6%AD%A2>)をご参照ください。
:::

Next.jsはコンセプトとして、**デフォルトで高いパフォーマンス**と**優れた開発者体験**を重視しています。App RouterはCache 1stな設計により、「デフォルトで高いパフォーマンス」を実現していますが、Cacheにまつわる「優れた開発者体験」については開発チームとユーザーコミュニティで大きな意見の隔たりが目立ちました。

### 当初の設計: 暗黙的なCache

App Routerは当初Cacheはデフォルトで有効で開発者ができるだけ気にしなくていいような形を目指し、一部のAPIを利用することで暗黙的にOpt-outされるよう設計されていました。しかし実際には、開発者にとって「デフォルトで有効」なことと「暗黙的にOpt-out」は開発者に多くの混乱を招きました。

WebアプリケーションにおいてCacheできないものを扱うことは非常に一般的なため、デフォルトで有効な以上開発者は気にするしかなく、一方でOpt-outが暗黙的なために多くの知見を必要としました。

### 問題の特定と改善

このようなコミュニティからのフィードバックを経て、Next.js開発チームは慎重に検討を重ねながら少しづつ改善を重ね、v13~15でCacheの開発者体験は確実によくなりました。以下はv13~15で行われた改善の例です。

- `fetch()`のデフォルトCache廃止
- Cacheに関するドキュメントの追加、改善
- Router Cacheに関する`staleTimes`オプションとデフォルト値の変更

これらの改善は確実に開発者体験を改善しましたが、一方でこれらの改善は特定の問題に対する場当たり的な対応のため、Opt-out型の設計に起因すると考えられる根本的な問題は解消されておらず、依然としてCacheには一定の複雑さが伴いました。

### Next.js開発チームのジレンマ

根本的な課題がOpt-out型の設計に起因するならば、解決策はOpt-in型に変更するしかありません。しかしこのような変更は大きな破壊的変更を伴い、何より「デフォルトで高いパフォーマンス」というコンセプトに反する変更になりえます。しかし一方で、何も変えなければ「優れた開発者体験」が犠牲になり続ける結果となります。Next.js開発チームは非常に困難なジレンマに直面しました。

この非常に困難なジレンマに取り組んで再設計された世界観こそが、**Cache Components**です。

## 設計・プラクティス

Cache Componentsは、従来のNext.jsのOpt-out型なCacheをOpt-in型に再設計した世界観です。そのため、Cacheに関する考え方やAPIが従来とは大きく異なることが特徴です。

Cache Componentsはこれまで実験的機能として開発されていたPPR・DynamicIO・`"use cache"`などを統合したものです。

### PPR(Partial Pre-Rendering)

**PPR**はページをStatic Renderingしつつ、一部をDynamic Renderingにすることができる、まさに部分的にPre-Renderingする手法です。

PPRに関する詳細な説明は筆者の過去の記事[「PPR - pre-rendering新時代の到来とSSR/SSG論争の終焉」↗︎](https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering#ppr%E3%81%A8%E3%81%AF)で解説しています。以下はこの記事の一部抜粋になります。

> PPRはStreaming SSRをさらに進化させた技術で、**ページをstatic renderingとしつつ、部分的にdynamic renderingにする**ことが可能なレンダリングモデルです。SSG・ISRのページの一部にSSRな部分を組み合わせられるようなイメージ、あるいはStreaming SSRのスケルトン部分をSSG/ISRにするイメージです。[公式の説明](https://rc.nextjs.org/learn/dashboard-app/partial-prerendering#what-is-partial-prerendering)よりECサイトの商品ページの例を拝借すると、以下のような構成が可能になります。
>
> ![ppr shell](/images/nextjs-partial-pre-rendering/ppr-shell.png)
>
> 商品ページ全体やナビゲーションはstatic renderingで静的化され、一方カートやレコメンド情報といったユーザーごとに異なるUI部分はdynamic renderingにすることができます。もちろん商品情報自体が更新されることもあるでしょうが、この例では必要に応じてrevalidateすることを想定しています。

以降Static Renderingされたページ部分のことを**Static Shell**、Dynamic Renderingな部分のことを**Dynamic Content**と呼称します。

### Dynamic IO

**Dynamic IO**は文字通り、Next.jsにおける動的I/O処理の扱いを大きく変更するもので、動的I/O処理を扱うには`<Suspense>`もしくは後述の`"use cache"`が必要^[Dynamic APIsはリクエスト時の情報を基本としているため、`"use cache"`することはできません。]になります。具体的には、以下のような処理を扱う場合に`<Suspense>`が必要になります。

- データフェッチ: `fetch()`やDBアクセスなど
- [Dynamic APIs](https://nextjs.org/docs/app/guides/caching#dynamic-apis): `headers()`や`cookies()`など
- Next.jsがラップするモジュール: `Date`、`Math`、Node.jsの`crypto`モジュールなど
- 任意の非同期関数(マイクロタスクを除く)

Cache ComponentsではPPRとDynamic IOを統合します。PPRはユニークで高いパフォーマンスが期待できる戦略ですが、Static ShellとDynamic Contentの境界を強く意識する必要があります。Dynamic IOでは上記のAPIをStatic Shell内で扱えないことで、境界への意識と予期せぬパフォーマンス劣化を防ぐことを目指しています。

### `"use cache"`

`"use cache"`は、コンポーネントや関数、もしくはファイルがCache可能であることを宣言するディレクティブです。`"use client"`や`"use server"`は[クライアントとサーバーのバンドル境界](./part_2_bundle_boundary)を宣言するものですが、`"use cache"`は**Cacheの境界**を宣言するものです。`"use cache"`が宣言されたコンポーネントや関数はbuild時に呼び出される可能性があり、これによりStatic Shellに動的データやコンポーネントを含むことができます。

`"use cache"`に関する詳細な使い方は後述の["use cache"とCache制御関数](./use_cache_directive)で解説します。

### その他: `<Activity>`

Cache Componentsを有効にすると、Next.jsは[`<Activity>`](https://ja.react.dev/reference/react/Activity)を使ってRouteを保護します。これまでは`useState()`によるコンポーネントの状態はページ遷移時のアンマウントで破棄されていましたが、Cache Componentsではアンマウントではなく`<Activity>`でRouteを保持する^[執筆時点では[最大3つまで](https://github.com/vercel/next.js/pull/77992)Routeを保持します。]ため、Route間の遷移で状態が復元されます。

これは特にブラウザバック・フォワード時に状態が復元されて見えるので、UX的に望ましい状態だと考えられます。

## トレードオフ

### `<Activity>`≠BF Cache

`<Activity>`によるブラウザバック時の体験向上は望ましい体験ですが、これはいわゆるブラウザの[BF Cache(Back Forward Cache)](https://web.dev/i18n/ja/bfcache/#bfcache%E3%81%AE%E5%9F%BA%E6%9C%AC)とは少々異なる点があります。管理する単位がRoute単位であって、履歴エントリー単位ではないので、ページA -> ページB -> ページAのように、同一URLへブラウザバックではなくリンクから遷移しても状態が復元されてしまいます。これはMPAにおけるブラウザ遷移体験とは異なる体験です。

また、前述の通り執筆時現在では最大3つまでしかRouteを保持できない点もUX上の大きな欠点です。

このようなトレードオフは、[nuqs](https://nuqs.dev/)などを利用してURLに状態を保存するか、もしくは筆者がメンテナンスしてる[location-state](https://github.com/recruit-tech/location-state)を利用することで解消することができます。location-stateは履歴エントリー単位での状態の復元をサポートするライブラリです。詳細は以下記事をご参照ください。

https://zenn.dev/akfm/articles/location-state
