---
title: "Cache Componentsの世界観"
---

## 要約

Cache Componentsは[PPR↗︎](https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering)とCacheの改善^[参考記事: [Next.js Cache回想↗︎](https://zenn.dev/akfm/articles/nextjs-caching-history)]により、Next.jsにおける重要なコンセプトである**デフォルトで高いパフォーマンス**と**優れた開発者体験**の両立を目指したものです。

Cache Componentsにおいてデータフェッチなど動的な処理を扱う際に、開発者は`<Suspense>`か`"use cache"`かどちらかを選択する必要があります。

## 背景

:::message
以下は筆者の過去記事[Next.js Cache回想↗︎](https://zenn.dev/akfm/articles/nextjs-caching-history)と重複します。より詳細な内容を知りたい方はこちらの記事をご参照ください。
:::

Next.jsはコンセプトとして、**デフォルトで高いパフォーマンス**と**優れた開発者体験**を重視しています。App RouterはCache 1stな設計により、デフォルトで高いパフォーマンスを実現していますが、Cacheの設計をはじめとする開発者体験については、開発チームとユーザーコミュニティで大きな意見の隔たりがありました。

### 当初の設計: 暗黙的で積極的なCache

App Routerは当初、Cacheをデフォルトで積極的に活用しつつ開発者ができるだけ気にしなくていいような形を目指し、一部のAPIを利用することで暗黙的にCacheがOpt-outされるような設計がされていました。しかし実際には、「デフォルトで有効なこと」と「暗黙的にOpt-outする設計」が開発者に多くの混乱を招きました^[[_Deep Dive: Caching and Revalidating_](https://github.com/vercel/next.js/discussions/54075)]。

WebアプリケーションにおいてCacheできないものを扱うことは非常に一般的ですが、Next.jsではデフォルトで有効なために開発者はCacheを意識しなければならず、しかしOpt-outが暗黙的なため、結果的に「意識しなければならないのに分かりづらい」という状況に陥る開発者が多発し、多くの混乱を生んだのだと考えられます。

### 問題の特定と改善

このようなコミュニティからのフィードバックを経て、Next.js開発チームは慎重に検討を重ねながら少しづつ改善を重ね、v13~15でCacheの開発者体験は確実によくなりました。以下はv13~15で行われた改善の例です。

- `fetch()`のデフォルトCache廃止
- Cacheに関するドキュメントの追加、改善
- Router Cacheの生存期間に関するオプションの追加

これらの改善は確実に開発者体験を改善しましたが、一方でこれらの改善は特定の問題に対する対症療法的な対応のため、Opt-out型の設計に起因する根本的な問題は解消されず、依然としてCacheには一定の複雑さが伴いました。

### Next.js開発チームのジレンマ

根本的な問題がOpt-out型の設計に起因するならば、解決策はOpt-in型に変更するしかありません。しかしこのような変更は大きな破壊的変更を伴う上、デフォルトで高いパフォーマンスというコンセプトに反する変更になりえます。しかし一方で、何も変えなければ優れた開発者体験が犠牲になり続ける結果となります。Next.js開発チームは非常に困難なジレンマに直面しました。

この非常に困難なジレンマに取り組んで再設計された世界観こそが、**Cache Components**です。

:::message
Next.js開発チームがCache Componentsに至る開発プロセスは非常に丁寧な検討と検証の積み重ねが見てとれます。筆者の私見は[Next.js Cache回想#私見↗︎](https://zenn.dev/akfm/articles/nextjs-caching-history#%E7%A7%81%E8%A6%8B)にて述べているので、ご参照ください。
:::

## 設計・プラクティス

Cache Componentsは従来のNext.jsのOpt-out型なCacheをOpt-in型に再設計したものであり、これまで実験的機能として開発されていたPPR・DynamicIO・`"use cache"`などを統合したものです。

### PPR(Partial Pre-Rendering)

**PPR**はページをStatic Renderingしつつ、一部をDynamic Renderingにすることができる、まさに部分的にPre-Renderingする手法です。

PPRに関する詳細な説明は筆者の過去の記事[PPR - pre-rendering新時代の到来とSSR/SSG論争の終焉↗︎](https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering#ppr%E3%81%A8%E3%81%AF)で解説しています。以下はこの記事の一部抜粋になります。

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

Cache ComponentsではPPRとDynamic IOを統合します。PPRはユニークで高いパフォーマンスが期待できる戦略ですが、従来のようにRouteのどこでも動的I/O処理を扱うことができる状態だと、Static ShellとDynamic Contentの境界が分かりづらいという課題がありました。

```tsx
export default async function Page({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  // 🤔`<Post />`はStatic or Dynamic?
  // 🤔`<Comments />`はStatic or Dynamic?
  return (
    <>
      <Post slug={slug} />
      <Comments slug={slug} />
    </>
  );
}
```

一方Dynamic IOでは動的I/O処理をStatic Shell内で扱えないため、Static ShellとDynamic Contentの境界が意識しやすくなり、またパフォーマンス劣化を防ぎやすい設計になりました。

```tsx
export default async function Page({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  // ✅`<Post />`はStatic
  // ✅`<Comments />`はDynamic
  return (
    <>
      <Post slug={slug} />
      <Suspense fallback={<>Loading...</>}>
        <Comments slug={slug} />
      </Suspense>
    </>
  );
}
```

### `"use cache"`

`"use cache"`は、コンポーネントや関数、もしくはファイルがCache可能であることを宣言するディレクティブです。`"use client"`や`"use server"`は[クライアントとサーバーのバンドル境界](./part_2_bundle_boundary)を宣言するものですが、`"use cache"`は**Cacheの境界**を宣言するものです。`"use cache"`が宣言されたコンポーネントや関数はbuild時に呼び出される可能性があり、これによりStatic Shellに動的データやコンポーネントを含むことができます。

![RSCの世界観とNext.jsのCache](/images/nextjs-caching-history/cache-door.png)
_RSCの世界観とNext.jsのCache_

`"use cache"`に関する詳細な使い方は後述の["use cache"とCache制御関数](./use_cache_directive)で解説します。

### その他: `<Activity>`

Cache Componentsを有効にすると、Next.jsはRouteに対し[`<Activity>`](https://ja.react.dev/reference/react/Activity)を使用します。従来はページ遷移毎にRouteがアンマウントされ、`useState()`による状態も破棄されていましたが、Cache Componentsでは`<Activity>`の`"visible"`と`"hidden"`を切り替えて遷移することがあるため、保持されてるRoute^[執筆時点では[最大3つまで](https://github.com/vercel/next.js/pull/77992)Routeを保持します。]コンポーネントの状態が保持されます。これにより、ブラウザバックやブラウザフォワード時に状態が復元されて見えることがあります。

ただし、Next.jsはRouteを履歴エントリー単位で管理していないため、いわゆる[BF Cacheとは異なる挙動](#activitybf-cache)を示すことに注意が必要です。

## トレードオフ

### `<Activity>`≠BF Cache

`<Activity>`によるブラウザバック時の体験向上は望ましい体験ですが、これはいわゆるブラウザの[BF Cache(Back Forward Cache)](https://web.dev/i18n/ja/bfcache/#bfcache%E3%81%AE%E5%9F%BA%E6%9C%AC)とは少々異なる点があります。管理する単位がRoute単位であって、履歴エントリー単位ではないので、ページA -> ページB -> ページAのように、同一URLへブラウザバックではなくリンクから遷移しても、状態が復元されて見えてしまいます。これはMPAにおけるブラウザ遷移体験とは異なる体験です。また、執筆時現在の仕様では最大3つまでしかRouteを保持できない大きな違いです。

このようなトレードオフは、[nuqs](https://nuqs.dev/)などを利用してURLに状態を保存するか、もしくは筆者がメンテナンスしてる[location-state](https://github.com/recruit-tech/location-state)を利用することで解消することができます。location-stateは履歴エントリー単位での状態の復元をサポートするライブラリです。詳細は以下記事をご参照ください。

https://zenn.dev/akfm/articles/location-state
