---
title: "[Experimental] Dynamic IO"
---

## 要約

Next.jsは現在、キャッシュの仕組みの大幅な刷新に取り組んでいます。これにより、本書で紹介してきたキャッシュに関する知識の多くは**過去のものとなる可能性**があります。

これからのNext.jsでキャッシュがどう変わるのか理解して、将来の変更に備えましょう。

:::message alert
本章で紹介するDynamic IOは執筆時点（v15.1.x）では実験的機能で、利用するには`canary`バージョンが必要となります。
:::

## 背景

[_第3部_](./part_3)ではApp Routerにおけるキャッシュの理解が重要であるとし、ここまで解説してきましたが、多層のキャッシュや複数の概念が登場し、難しいと感じた方も多かったのではないでしょうか。実際、App Router登場当初から現在に至るまでキャッシュに関する批判的意見は多く、現在のNext.jsにおける最も大きなネガティブ要素と言っても過言ではありません。

https://github.com/vercel/next.js/discussions/54075

一方Next.jsは、**デフォルトで高いパフォーマンス**を実現できることを重視したフレームワークです。開発者のフィードバックを優先してキャッシュは全てオプトインにすることはデフォルトで高いパフォーマンスというコンセプトを蔑ろにすることにつながってしまうため、キャッシュに関する批判への対応は非常に難しい問題でした。

## 設計・プラクティス

**Dynamic IO**は文字通り、Next.jsにおける動的I/O処理の振る舞いを大きく変更するものです。

https://nextjs.org/docs/app/api-reference/config/next-config-js/dynamicIO

具体的には、動的I/O処理とは以下を指します。

- データフェッチ: `fetch()`やDBアクセスなど
- [Dynamic APIs](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-apis): `headers()`や`cookies()`など
- Next.jsがラップするモジュール: `Date`、`Math`、Node.jsの`crypto`モジュールなど
- 任意の非同期関数(マイクロタスクを除く)

Dynamic IOでは、これらの処理を含む場合には`<Suspense>`でレンダリングを遅延させるか`"use cache"`でキャッシュするか選択する必要があり、デフォルトではこれらの動的I/O処理は扱うことはできません。

これは、デフォルトキャッシュがもたらした混乱に対し、デフォルトでキャッシュするかしないかではなく、**明示的な選択を強制する**というアプローチによって、高いパフォーマンスを実現しやすい形をとりつつも開発者の混乱を解消することを目指したもので、筆者はシンプルかつ柔軟な設計だと評価しています。このようなアプローチが可能となったのは、`"use cache"`による**キャッシュ境界を指定可能にする発明**が大きく寄与しています。

### `<Suspense>`境界内の遅延実行

Dynamic IOで動的I/O処理を扱いたい場合、1つ目の選択肢として挙げられるのが`<Suspense>`によってStreamingで段階的にレンダリング結果を配信する方法です。

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

上記の場合、ユーザーには`Your Profile`というタイトルと`<Loading />`が表示され、その後に`<Profile>`の内容が表示されます。`<Page>`はリクエストごとにレンダリングされますが、動的I/O処理を含まないため非常に高速な表示が期待できます。

### `"use cache"`によるキャッシュ

Dynamic IOでは、`"use client"`のようにキャッシュの境界を`"use cache"`で指定します。前述の通り、デフォルトではキャッシュされないので`"use cache"`による明示的なオプトインが必要です。

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

キャッシュは通常キーが必要になりますが、`"use cache"`ではコンパイラがキャッシュのキーを自動で生成します。引数や参照してる変数などをキーとして認識されますが、`children`のような直接シリアル化できないものは**キーに含まれません**。これにより、`children`が参照レベルで同一のものを渡さないとキャッシュが効かない、といったことが発生しないよう設計されています。

`"use cache"`はファイルやコンポーネントを含む関数レベルで指定することができます。ファイルに指定した場合には、すべての`export`される関数に対し`"use cache"`が適用されます。

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

### キャッシュの詳細な指定

`"use cache"`の世界では従来より自由度の高いキャッシュ戦略が可能になります。具体的には、キャッシュのタグや有効期間の指定方法がより柔軟になりました。

従来は`fetch()`のオプションでタグを指定するなどしていたため、データフェッチ後にタグをつけることができませんでしたが、Dynamic IOでは`cacheTag()`で関数にタグを付与することができるので、より柔軟な指定が可能になりました。

```tsx
import { unstable_cacheTag as cacheTag } from "next/cache";

async function getBlogPosts(page: number) {
  "use cache";

  const posts = await fetchPosts(page);
  posts.forEach((post) => {
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

  cacheLife("minutes");

  const posts = await fetchPosts(page);
  return posts;
}
```

## トレードオフ

### Experimental期間

TBW

### キャッシュの永続化

TBW

### キャッシュに関する制約

TBW

### `<Suspense>`利用時には`fallback`のレンダリングが必至

TBW
