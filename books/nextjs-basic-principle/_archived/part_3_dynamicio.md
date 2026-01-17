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

[_第3部_](./part_3)ではApp Routerにおけるキャッシュの理解が重要であるとし、解説してきましたが、多層のキャッシュや複数の概念が登場し、難しいと感じた方も多かったのではないでしょうか。実際、App Router登場当初から現在に至るまでキャッシュに関する批判的意見^[[Next.jsのDiscussion↗︎](https://github.com/vercel/next.js/discussions/54075)では、批判的な意見や改善要望が多く寄せられました。]は多く、現在のNext.jsにおける最も大きな課題の一つと言えるでしょう。

開発者の混乱を解決する最もシンプルな方法は、キャッシュをデフォルトで有効化する形をやめて、オプトイン方式に変更することです。ただし、Next.jsは**デフォルトで高いパフォーマンス**を実現することを重視したフレームワークであるため、このような変更はコンセプトに反します。

Next.jsは、キャッシュにまつわる混乱の解決とデフォルトで高いパフォーマンスの両立という、非常に難しい課題に取り組んできました。

## 設計・プラクティス

**Dynamic IO**は、前述の課題に対しNext.jsコアチームが検討を重ねて生まれた1つの解決案で、文字通りNext.jsにおける動的I/O処理の振る舞いを大きく変更するものです。

https://nextjs.org/docs/app/api-reference/config/next-config-js/dynamicIO

ここで言う動的I/O処理にはデータフェッチや`headers()`や`cookies()`などの[Dynamic APIs↗︎](https://nextjs.org/docs/app/guides/caching#dynamic-apis)が含まれ^[動的I/O処理には`Date`、`Math`といったNext.jsが拡張してるモジュールや、任意の非同期関数なども含まれます]、Dynamic IOではこれらの処理を含む場合、以下いずれかの対応が必要となります。

- **`<Suspense>`**: 非同期コンポーネントを`<Suspense>`境界内に配置し、Dynamic Renderingにする
- **`"use cache"`**: 非同期関数や非同期コンポーネントに`"use cache"`を指定して、Static Renderingにする

ここで重要なのは、Dynamic IO以前のように非同期処理を自由に扱えるわけではなく、**上記いずれかの対応が必須となる**点です。これは、Dynamic IO以前のデフォルトキャッシュがもたらした混乱に対し明示的な選択を強制することで、高いパフォーマンスを実現しやすい形をとりつつも開発者の混乱を解消することを目指したもので、筆者はシンプルかつ柔軟な設計だと評価しています。

### `<Suspense>`によるDynamic Rendering

ECサイトのカートやダッシュボードなど、リアルタイム性や細かい認可制御などが必要な場面では、キャッシュされないデータフェッチが必須です。これらで扱うような非常に動的なコンポーネントを実装する場合、Dynamic IOでは`<Suspense>`境界内で動的I/O処理を扱うことができます。`<Suspense>`境界内はDynamic IO以前同様、Streamingで段階的にレンダリング結果が配信されます。

```tsx
async function Profile({ id }: { id: string }) {
  const user = await getUser(id);

  return (
    <>
      <h2>{user.name}</h2>
      ...
    </>
  );
}

export default async function Page() {
  return (
    <>
      <h1>Your Profile</h1>
      <Suspense fallback={<Loading />}>
        <Profile />
      </Suspense>
    </>
  );
}
```

上記の場合、ユーザーにはまず`Your Profile`というタイトルと`fallback`の`<Loading />`が表示され、その後に`<Profile>`の内容が`fallback`に置き換わって表示されます。これは`<Profile>`が並行レンダリングされ、完了次第ユーザーに配信されるためにこのような挙動になります。

### `"use cache"`によるStatic Rendering

一方、商品情報やブログ記事などのキャッシュ可能なコンポーネントを実装する場合、Dynamic IOではキャッシュしたい関数やコンポーネントに`"use cache"`を指定します。`"use cache"`はStatic Renderingな境界を生成し、子孫コンポーネントまで含めてStatic Renderingとなります。

以下は前述の`<Profile>`に`"use cache"`を指定してStatic Renderingにする例です。

```tsx
async function Profile({ id }: { id: string }) {
  "use cache";

  const user = await getUser(id);

  return (
    <>
      <h2>{user.name}</h2>
      ...
    </>
  );
}
```

キャッシュは通常キーが必要になりますが、`"use cache"`ではコンパイラがキャッシュのキーを自動で識別します。具体的には、引数やクロージャが参照してる外側のスコープの変数などをキーとして認識します。一方で、`children`のような直接シリアル化できないものは**キーに含まれません**。これにより、`"use cache"`のキャッシュ境界は`"use client"`同様、[_Compositionパターン_](./part_2_composition_pattern)が適用できます。

より詳細な説明についてはNext.jsの公式ブログに解説記事があるので、こちらをご参照ください。

https://nextjs.org/blog/composable-caching#how-does-it-work

なお、`"use cache"`はコンポーネントを含む関数やファイルレベルで指定することができます。ファイルに指定した場合には、すべての`export`される関数に対し`"use cache"`が適用されます。

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

`"use cache"`を使ったキャッシュでは、Dynamic IO以前より自由度の高いキャッシュ戦略が可能となります。具体的には、キャッシュのタグや有効期間の指定方法がより柔軟になりました。

Dynamic IO以前は`fetch()`のオプションでタグを指定するなどしていたため、データフェッチ後にタグをつけることができませんでしたが、Dynamic IOでは[`cacheTag()`↗︎](https://nextjs.org/docs/app/api-reference/functions/cacheTag)でタグを指定するため、`fetch()`後にタグを付与するなど柔軟な指定が可能になりました。

```tsx
import { unstable_cacheTag as cacheTag } from "next/cache";

async function getBlogPosts(page: number) {
  "use cache";

  const posts = await fetchPosts(page);
  posts.forEach((post) => {
    // 🚨Dynamic IO以前はタグを`fetch()`時に指定する必要があったため、`posts`などを参照できなかった
    cacheTag("blog-post-" + post.id);
  });

  return posts;
}
```

同様に、キャッシュの有効期限も[`cacheLife()`↗︎](https://nextjs.org/docs/app/api-reference/functions/cacheLife)で指定できます。`cacheLife()`は`"minutes"`などの`profile`と呼ばれる時間指定に関するラベル文字列を引数にとり、`"use cache"`指定時に`cacheLife()`を明示的に指定することを推奨されています。

```tsx
import { unstable_cacheLife as cacheLife } from "next/cache";

async function getBlogPosts(page: number) {
  "use cache";

  cacheLife("minutes");

  const posts = await fetchPosts(page);
  return posts;
}
```

`profile`の値はStale・Revalidate・Expireの3つの振る舞いに対応し、カスタマイズも可能です。

- Stale: クライアントサイドのキャッシュ期間。
- Revalidate: サーバー上でキャッシュを更新する頻度。Revalidate中は古い値が提供される場合があります。
- Expire: 値が古いままでいられる最大期間。Revalidateより長くする必要があります。

以下はデフォルトで指定可能な`profile`です。

| `profile` | Stale     | Revalidate | Expire         |
| --------- | --------- | ---------- | -------------- |
| `default` | undefined | 15 minutes | INFINITE_CACHE |
| `seconds` | undefined | 1 second   | 1 minute       |
| `minutes` | 5 minutes | 1 minute   | 1 hour         |
| `hours`   | 5 minutes | 1 hour     | 1 day          |
| `days`    | 5 minutes | 1 day      | 1 week         |
| `weeks`   | 5 minutes | 1 week     | 1 month        |
| `max`     | 5 minutes | 1 month    | INFINITE_CACHE |

### `<Suspense>`と`"use cache"`の併用

`<Suspense>`と`"use cache"`は併用が可能である点も、Dynamic IO以前と比較して非常に優れている点です。以下のように、動的な要素とキャッシュ可能な静的な要素を組み合わせることができます。

```tsx
export default function Page() {
  return (
    <>
      {/* Static Rendering */}
      ...
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

ただし、`"use cache"`は`"use client"`同様境界を示すものなので、`children`を除きDynamic Renderingなコンポーネントを含むことができません。これは`<Suspense>`も例外ではないので、以下のような実装はできません。

```tsx
async function StaticComponent() {
  "use cache";

  return (
    <>
      ...
      {/* 🚨Dynamic Renderingなコンポーネントは含むことができない */}
      <Suspense>
        <DynamicComponent />
      </Suspense>
    </>
  );
}
```

Client Components同様慣れが必要な部分になるので、以下のルールをしっかり覚えておきましょう。

- Dynamic RenderingはStatic Renderingを含むことができる
- Static RenderingはDynamic Renderingを`children`でなら含むことができる

## トレードオフ

### Experimental

Dynamic IOは、本書執筆時点でまだexperimentalな機能という位置付けです。そのため、Dynamic IOを利用するにはNext.jsの`canary`バージョンと、`next.config.ts`に以下の設定を追加する必要があります。

```ts :next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  experimental: {
    dynamicIO: true,
  },
};

export default nextConfig;
```

### キャッシュの永続化

Dynamic IOにおけるキャッシュの永続化は`next.config.ts`を通じてカスタマイズ可能ですが、Dynamic IO以前からある[Custom Cache Handler↗︎](https://nextjs.org/docs/app/api-reference/config/next-config-js/incrementalCacheHandlerPath)とは別物になります。少々複雑ですが、Dynamic IO以前のものが`cacheHandler`で設定できたのに対し、Dynamic IOのキャッシュハンドラーは`experimental.cacheHandlers`で設定します。

```ts :next.config.ts
import type { NextConfig } from "next";
import path from "path";

const nextConfig: NextConfig = {
  experimental: {
    dynamicIO: true,
    cacheHandlers: {
      // ref: https://github.com/vercel/next.js/blob/c228a6e65d4b7973aa502544f9f8e025a6f97066/packages/next/src/server/config-shared.ts#L240-L245
      default: path.join(import.meta.dirname, "..."),
      remote: path.join(import.meta.dirname, "..."),
      static: path.join(import.meta.dirname, "..."),
    },
  },
};

export default nextConfig;
```

:::message
執筆時現在、これらの利用方法などもまだドキュメントが見当たらないので、Next.jsコアチームの対応を待つ必要があります。
:::

### キャッシュに関する制約

`"use cache"`のキャッシュのキーは自動でコンパイラが識別してくれるので非常に便利ですが、一方[シリアル化↗︎](https://ja.react.dev/reference/rsc/use-server#serializable-parameters-and-return-values)不可能なものはキャッシュのキーに含まれないため注意が必要です。下記のように関数を引数に取る場合は、`"use cache"`を使用しない方が意図しない動作を防ぐことができます。

```tsx
async function cachedFunctionWithCallback(callback: () => void) {
  "use cache";

  // ...
}
```

また、`"use cache"`を指定した非同期関数の戻り値は必ずシリアル化可能である必要があります。
