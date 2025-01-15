---
title: "[Experimental] Dynamic IO"
---

## 要約

Next.jsは現在、キャッシュの大幅な刷新に取り組んでいます。これにより、本書で紹介してきたキャッシュに関する知識の多くは**過去のものとなる可能性**があります。

これからのNext.jsでキャッシュがどう変わるのか理解して、将来の変更に備えましょう。

:::message alert
本章で紹介するDynamic IOは執筆時点（v15.1.x）では実験的機能で、利用するには`canary`バージョンが必要となります。
:::

## 背景

[_第3部_](./part_3)ではApp Routerにおけるキャッシュの理解が重要であるとし、解説してきましたが、多層のキャッシュや複数の概念が登場し、難しいと感じた方も多かったのではないでしょうか。実際、App Router登場当初から現在に至るまでキャッシュに関する批判的意見^[[Next.jsのDiscussion](https://github.com/vercel/next.js/discussions/54075)では、批判的な意見や改善要望が多く寄せられました。]は多く、現在のNext.jsにおける最も大きな課題の一つと言えるでしょう。

開発者の混乱を解決する最もシンプルな方法は、キャッシュをデフォルトで有効化する形をやめて、オプトイン方式に変更することです。ただし、Next.jsは**デフォルトで高いパフォーマンス**を実現することを重視したフレームワークであるため、このような変更はコンセプトに反します。

Next.jsは、キャッシュにまつわる混乱の解決とデフォルトで高いパフォーマンスの両立という、非常に難しい課題に取り組んできました。

## 設計・プラクティス

**Dynamic IO**は、前述の課題に対しNext.jsコアチームが検討を重ねて生まれた1つの解決案で、文字通りNext.jsにおける動的I/O処理の振る舞いを大きく変更するものです。

https://nextjs.org/docs/app/api-reference/config/next-config-js/dynamicIO

ここで言う動的I/O処理にはデータフェッチや`headers()`や`cookies()`などの[Dynamic APIs](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-apis)が含まれ^[動的I/O処理には`Date`、`Math`といったNext.jsが拡張してるモジュールや、任意の非同期関数なども含まれます]、Dynamic IOではこれらの処理を含む場合、以下いずれかの対応が必要となります。

- **`<Suspense>`**: 非同期コンポーネントを`<Suspense>`境界内に配置し、Dynamic Renderingにする
- **`"use cache"`**: 非同期関数や非同期コンポーネントに`"use cache"`を指定して、Static Renderingにする

ここで重要なのは、従来のように非同期処理を自由に扱えるわけではなく、**上記いずれかの対応が必須となる**点です。これは、従来のデフォルトキャッシュがもたらした混乱に対し明示的な選択を強制することで、高いパフォーマンスを実現しやすい形をとりつつも開発者の混乱を解消することを目指したもので、筆者はシンプルかつ柔軟な設計だと評価しています。

### `<Suspense>`によるDynamic Rendering

ECサイトのカートやダッシュボードなど、リアルタイム性や細かい認可制御などが必要な場面では、キャッシュされないデータフェッチが必須です。これらで扱うような非常に動的なコンポーネントを実装する場合、Dynamic IOでは`<Suspense>`境界内で動的I/O処理を扱うことができます。`<Suspense>`境界内は従来同様、Streamingで段階的にレンダリング結果が配信されます。

```tsx
async function Profile({ id, children }: { id: string; children: ReactNode }) {
  const user = await getUser(id);

  return (
    <>
      <h2>{user.name}</h2>
      {children}
    </>
  );
}

export default async function Page() {
  return (
    <>
      <h1>Your Profile</h1>
      <Suspense fallback={<Loading />}>
        <Profile>...</Profile>
      </Suspense>
    </>
  );
}
```

上記の場合、ユーザーにはまず`Your Profile`というタイトルと`fallback`の`<Loading />`が表示され、その後に`<Profile>`の内容が`fallback`に置き換わって表示されます。これは`<Profile>`が並行レンダリングされ、完了次第ユーザーに配信されるためにこのような挙動になります。

### `"use cache"`によるStatic Rendering

一方、商品情報やブログ記事などのキャッシュ可能な要素を実装する場合、Dynamic IOではキャッシュしたい関数やコンポーネントなどの境界に`"use cache"`を指定します。`"use cache"`のキャッシュ境界は`"use client"`同様、Compositionパターンが利用できるので`children`を渡すことも可能です。

以下は前述の`<Profile>`をキャッシュする例です。

```tsx
async function Profile({ id, children }: { id: string; children: ReactNode }) {
  "use cache";

  const user = await getUser(id);

  return (
    <>
      <h1>{user.name}</h1>
      {children}
    </>
  );
}
```

キャッシュは通常キーが必要になりますが、`"use cache"`ではコンパイラがキャッシュのキーを自動で生成します。引数や参照してる変数などをキーとして認識されますが、`children`のような直接シリアル化できないものは**キーに含まれません**。これにより、`children`のようなコンポーネントの計算に直接影響がないものを含んでいても、キャッシュがしっかりと有効になるように設計されています。また、Client Components同様`"use cache"`によって生成されるキャッシュ境界も[_Compositionパターン_](./part_2_composition_pattern)が適用できます。

`"use cache"`はコンポーネントを含む関数やファイルレベルで指定することができます。ファイルに指定した場合には、すべての`export`される関数に対し`"use cache"`が適用されます。

```tsx
// File level
"use cache";

export default async function Page() {
  // ...
}

// Component level
export async function MyComponent() {
  "use cache";

  return <></>;
}

// Function level
export async function getData() {
  "use cache";

  const data = await fetch("/api/data");
  return data;
}
```

:::message
`"use cache"`が適用される関数は非同期関数である必要があります。
:::

### キャッシュの詳細な指定

`"use cache"`を使ったキャッシュでは、従来より自由度の高いキャッシュ戦略が可能となります。具体的には、キャッシュのタグや有効期間の指定方法がより柔軟になりました。

従来は`fetch()`のオプションでタグを指定するなどしていたため、データフェッチ後にタグをつけることができませんでしたが、Dynamic IOでは`cacheTag()`で関数にタグを付与することができるので、より柔軟な指定が可能になりました。

```tsx
import { unstable_cacheTag as cacheTag } from "next/cache";

async function getBlogPosts(page: number) {
  "use cache";

  const posts = await fetchPosts(page);
  posts.forEach((post) => {
    // 🚨従来は`fetch()`時に指定する必要があったため、`posts`などを参照できなかった
    cacheTag("blog-post-" + post.id);
  });

  return posts;
}
```

同様に、キャッシュの有効期限も`cacheLife()`で指定できます。

```tsx
import { unstable_cacheLife as cacheLife } from "next/cache";

async function getBlogPosts(page: number) {
  "use cache";

  cacheLife("minutes"); // 1min

  const posts = await fetchPosts(page);
  return posts;
}
```

### `<Suspense>`と`"use cache"`の併用

`<Suspense>`と`"use cache"`は併用が可能である点も、従来と比較して非常に優れている点です。以下のように、動的な要素とキャッシュ可能な静的な要素を組み合わせることができます。

```tsx
export default function Page() {
  return (
    <>
      ...
      {/* Static Rendering */}
      <Suspense fallback={<Loading />}>
        {/* Dynamic Rendering */}
        <DynamicComponent>
          {/* Static Rendering */}
          <StaticComponent />
        </DynamicComponent>
      </Suspense>
    </>
  );
}
```

## トレードオフ

### キャッシュの永続化

Dynamic IOにおけるキャッシュの永続化は`next.config.js`を通じてカスタマイズ可能ですが、従来からある[Custom Cache Handler](https://nextjs.org/docs/app/api-reference/config/next-config-js/incrementalCacheHandlerPath)とは別物になります。少々複雑ですが、従来のものが`cacheHandler`で設定できたのに対し、Dynamic IOのキャッシュハンドラーは`experimental.cacheHandlers`で設定します。

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    dynamicIO: true,
    cacheHandlers: {
      // ref: https://github.com/vercel/next.js/blob/c228a6e65d4b7973aa502544f9f8e025a6f97066/packages/next/src/server/config-shared.ts#L240-L245
      default: require.resolve("..."),
      remote: require.resolve("..."),
      static: require.resolve("..."),
    },
  },
};

module.exports = nextConfig;
```

執筆時現在、これらの利用方法などもまだドキュメントが見当たらないので、Next.jsコアチームの対応を待つ必要があります。

### キャッシュに関する制約

`"use cache"`のキャッシュのキーは自動でコンパイラが識別してくれるので、非常に便利ですが、一方[シリアル化](https://ja.react.dev/reference/rsc/use-server#serializable-parameters-and-return-values)不可能なものはキャッシュのキーに含まれないため注意が必要です。下記のように関数を引数に取る場合は、`"use cache"`を使用しない方が意図しない動作を防ぐことができます。

```tsx
async function cachedFunctionWithCallback(callback: () => void) {
  "use cache";

  // ...
}
```

また、`"use cache"`を指定した関数の戻り値は必ずシリアル化可能である必要があります。

### `<Suspense>`利用時には`fallback`のレンダリングが必至

Dynamic Renderingで動的I/O処理を扱う際には`<Suspense>`を利用する必要があるため、`fallback`のレンダリングが必至です。未指定の場合にも[Cumulative Layout Shift](https://web.dev/articles/cls?hl=ja)（CLS）が発生するため、`fallback`の指定を忘れないようにしましょう。
