---
title: "明示的で合成可能なCache戦略"
---

## 要約

Cache Componentsは[PPR↗︎](https://nextjs.org/blog/next-14#partial-prerendering-preview)・[Dynamic IO↗︎](https://nextjs.org/blog/our-journey-with-caching)・[`"use cache"`↗︎](https://nextjs.org/docs/app/api-reference/directives/use-cache)を統合したものです。Next.jsにおいて重要なコンセプトである**デフォルトで高いパフォーマンスと優れた開発者体験の両立**に基づき、従来大きな課題だったCache戦略を大きく変更しています。

従来Next.jsは暗黙的で多層のCache戦略を採用していましたが、Cache Componentsでは**明示的で合成可能なCache戦略**が重視されています。

## 背景

:::message
以下は筆者の過去記事[Next.js Cache回想↗︎](https://zenn.dev/akfm/articles/nextjs-caching-history)と重複します。より詳細な内容を知りたい方はこちらの記事をご参照ください。
:::

Next.jsは**デフォルトで高いパフォーマンスと優れた開発者体験の両立**をコンセプトとして重視しています。特に、App RouterはCache 1stな設計によりデフォルトで高いパフォーマンスを実現してきましたが、開発者体験についてはコミュニティから不評で、筆者自身も複雑性が高いと感じていました。

### 暗黙的で多層のCache

App Routerでは当初、Cacheをデフォルトで積極的に活用しつつ開発者ができるだけ気にしなくていいような形を目指し、一部のAPIを利用することで暗黙的にCacheがOpt-outされるような戦略が採用されていました。しかし、このような戦略は開発者に多くの混乱を招き、[Next.jsのDiscussion↗︎](https://github.com/vercel/next.js/discussions/54075)では批判的な意見や改善要望が数多く寄せられました。

WebアプリケーションにおいてCacheできないものを扱うことは非常に一般的ですが、Next.jsではデフォルトで有効なために開発者はCacheを意識しなければなりません。しかし、Opt-out型の設計によりCacheは暗黙的に有効であるため、「暗黙的なAPIを意識しなければならない」状況が多くの混乱を生んだものと考えられます。

### 小さな改善の積み重ね

このようなコミュニティからのフィードバックを経て、Next.js開発チームは慎重に検討を重ねながら少しずつ改善を重ね、v13\~15でCacheの開発者体験は確実によくなりました。以下はv13\~15で行われた改善の例です。

- `fetch()`のデフォルトCache廃止
- Cacheに関するドキュメントの追加^[当初は「開発者ができるだけ気にしなくていいような形」を目指していたため、Cacheに関するドキュメントも意図的に用意してなかったようです。]、改善
- Router Cacheの生存期間に関するオプションの追加

これらの改善は確実に開発者体験を改善しましたが、一方でこれらの改善は特定の問題に対する対症療法的な対応であり、暗黙的で多層なCache戦略に起因する根本的な複雑さや認知負荷は解消されませんでした。

### Next.js開発チームのジレンマ

根本的な問題が暗黙的で多層のCacheに起因するとすれば、最もシンプルな改善は明示的なCache戦略に変更することです。しかし、このような変更は大きな破壊的変更を伴うであろうことや、Next.jsのコンセプトであるデフォルトで高いパフォーマンスに反する変更になりえることが懸念されました。一方で何も変えなければ、もう一つ重要なコンセプトである優れた開発者体験が犠牲になり続ける結果となります。Next.js開発チームは非常に困難なジレンマに直面しました。

この非常に困難なジレンマに対し、様々な検討と検証を経て誕生したのが**Cache Components**です。

## 設計・プラクティス

Cache Componentsは実験的機能として開発されていたPPR・Dynamic IO・`"use cache"`を統合したもので、従来の暗黙的で多層のCache戦略から脱却すべく、**明示的で合成可能なCache戦略**を採用しています。

### PPR(Partial Pre-Rendering)

**PPR**はPartial Pre-Renderingの略称で、ページをStatic Renderingしつつ一部をDynamic Renderingにすることができる、文字通りPartial（部分的）にPre-Rendering（事前レンダリング）するレンダリングモデルです。

詳細な経緯や解説は筆者の過去の記事[PPR - pre-rendering新時代の到来とSSR/SSG論争の終焉↗︎](https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering#ppr%E3%81%A8%E3%81%AF)をご参照ください。ここでは一部説明を抜粋します。

> PPRはStreaming SSRをさらに進化させた技術で、**ページをstatic renderingとしつつ、部分的にdynamic renderingにする**ことが可能なレンダリングモデルです。SSG・ISRのページの一部にSSRな部分を組み合わせられるようなイメージ、あるいはStreaming SSRのスケルトン部分をSSG/ISRにするイメージです。[公式の説明](https://rc.nextjs.org/learn/dashboard-app/partial-prerendering#what-is-partial-prerendering)よりECサイトの商品ページの例を拝借すると、以下のような構成が可能になります。
>
> ![ppr shell](/images/nextjs-partial-pre-rendering/ppr-shell.png)

以降Static Renderingされたページの外郭を**Static Shell**、Dynamic Renderingなページの一部分のことを**Dynamic Content**と呼称します。

Next.jsはリクエストを受け取ると即座にStatic Shellを配信することで、TTFB（Time to First Byte）とFCP（First Contentful Paint）を高速化しています。特にVercelでは、Static Shellの配信をCDNが担うため、非常に高速な体験が期待できます^[執筆時点では、Vercel以外でStatic ShellをCDNから配信することは困難だと思われます。]。

### Dynamic IO

**Dynamic IO**は、動的I/O処理を扱う際に`<Suspense>`を必須とする**制約機能**です。従来Next.jsでは動的I/O処理を`<Suspense>`なしで扱うことができたため、動的I/O処理によってレンダリングがブロックされ、TTFBやFCPが低速になる可能性がありました。これを回避するため従来は積極的なCache活用がされていましたが、Dynamic IOでは制約により`<Suspense>`の外側^[Dynamic IOはPPRと組み合わせることが前提となっているため、`<Suspense>`境界の外側はStatic Shellとなります。]には動的I/O処理が含まれなくなるため、Static Shellのレンダリングがブロックされず、TTFBやFCPを高速に保つことができます。

具体的には、以下のような処理を扱う場合に`<Suspense>`が必要になります。

- データフェッチ: `fetch()`やDBアクセスなど
- [Runtime Data](https://nextjs.org/docs/app/getting-started/cache-components#runtime-data): `params`（`generateStaticParams()`で事前に解決される場合は除く）や`cookies()`など
- Next.jsがラップするモジュール: `Date`、`Math`、Node.jsの`crypto`モジュールなど
- 任意の非同期関数（マイクロタスクを除く）

```tsx
// ✅ Next.jsサーバーは即座にfallbackを含むUIを描画し始めることができる
export default async function Page({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  // ⚠️`generateStaticParams()`を定義してるので`params`はbuild時に解決されるが、
  // 未定義の場合はリクエスト時に解決される
  const { slug } = await params;
  // `<PostContainer />`はStatic
  // `<Comments />`はDynamic
  return (
    <>
      <PostContainer slug={slug} />
      <Suspense fallback={<>Loading...</>}>
        <CommentsContainer slug={slug} />
      </Suspense>
    </>
  );
}

export async function generateStaticParams() {
  return [{ slug: "1" }, { slug: "2" }];
}
```

従来は開発者がDynamic Renderingを意識するのに暗黙的なAPIと広範な実装把握が必要でしたが、Dynamic IOでは`<Suspense>`によりDynamic Renderingの境界が明確になりました。

なお、Cache ComponentsではPPRとDynamic IOを統合しているため、全てのRouteで`<Suspense>`の外側はStatic Shellとなります。

### `"use cache"`

`"use cache"`は、コンポーネントや関数が**Cache可能である**ことを宣言するディレクティブです。`"use client"`や`"use server"`は[クライアントとサーバーのバンドル境界](part_2_bundle_boundary)を宣言するものですが、`"use cache"`は**Cacheの境界**を宣言するものです。

![RSCの世界観とNext.jsのCache](/images/nextjs-caching-history/cache-door.png)
_RSCの世界観とNext.jsのCache_

`"use cache"`が宣言されたコンポーネントや関数はCache可能なものとして扱われるため、Static Shellに含まれる可能性があります。

:::message
Dynamic Contentから`"use cache"`な関数やコンポーネントが参照されることもあるため、`"use cache"`=必ずしもStatic Shellに含まれるとは限りません。
:::

### その他: `<Activity>`

Cache Componentsを有効にすると、Next.jsはRouteに対し[`<Activity>`↗︎](https://ja.react.dev/reference/react/Activity)を使用します。従来はページ遷移毎にRouteがアンマウントされ、`useState()`による状態も破棄されていましたが、Cache Componentsでは`<Activity>`の`"visible"`と`"hidden"`を切り替えて遷移するため、ブラウザバックもしくはブラウザフォワードでは、状態が復元されて見えることがあります。

ただし、厳密にはいわゆるBF Cacheと異なる振る舞いをする点に注意が必要です。

:::message
執筆時時点では未リリースですが、Cache Componentsにおける`<Activity>`の使用は実験的な機能として、[`experimental.cachedNavigations`フラグ](https://github.com/vercel/next.js/pull/90928)に分離されました。この変更は、v16.2のリリースに含まれる可能性があります。
:::

## トレードオフ

### `<Activity>`≠BF Cache

`<Activity>`によるブラウザバック時の体験向上は好ましいですが、これはいわゆる[BF Cache(Back Forward Cache)↗︎](https://web.dev/i18n/ja/bfcache/#bfcache%E3%81%AE%E5%9F%BA%E6%9C%AC)とは異なるものです。Next.jsが管理する単位はRoute単位であり**履歴エントリー単位ではない**ので、

1. ページA
2. ページB
3. ページA

のように、同一URLへブラウザバックではなくリンクから遷移しても、状態が復元されて見えてしまいます。これはMPAにおけるブラウザ遷移体験とは異なる体験です。また、執筆時現在の仕様では[最大3つまで↗︎](https://github.com/vercel/next.js/pull/77992)しかRouteの状態を保持できないのも大きな違いです。

このようなトレードオフは、[nuqs↗︎](https://nuqs.dev/)などを利用してURLに状態を保存するか、もしくは筆者らが開発してる[location-state↗︎](https://github.com/recruit-tech/location-state)を利用することで解消することができます。location-stateはまさしく、履歴エントリー単位での状態の復元をサポートするライブラリです。詳細は以下記事をご参照ください。

https://zenn.dev/akfm/articles/location-state
