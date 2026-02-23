---
title: "Cacheの境界と制御"
---

## 要約

`"use cache"`はCacheの境界を宣言するもので、Cacheの制御は制御関数によって行います。従来の設定による制御と比べると、制御関数は高い自由度とより詳細な設定が可能になっています。

## 背景

[明示的で合成可能なCache戦略](explicit_cache_composability)で解説したように、Cache Componentsでは`"use cache"`を関数やファイルで宣言することで関数やコンポーネントが**Cache可能である**ことを表明します。

```tsx
async function PostContainer({ slug }: { slug: string }) {
  "use cache";

  const post = await getPost({ slug });

  return (
    <main>
      <h1>{post.title}</h1>
      ...
    </main>
  );
}
```

これは、従来の暗黙的で多層のCache戦略とは全く異なるものです。

### 従来のCache境界と制御

従来のCacheは多層で、特にサーバーサイドにおいてはFull Route CacheとData Cacheの役割と相互影響などを考慮する必要がありました。

Full Route Cacheの制御は、以下のように[Route Segment Config↗︎](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config)で行うことが可能です。

```tsx
// layout.tsx | page.tsx
export const revalidate = 0; // 1以上でStatic Rendering
```

一方Data Cacheの制御は、`fetch()`の引数で制御が可能です。

```tsx
async function getPost({ slug }: { slug: string }) {
  return fetch(`https://...`, {
    next: {
      tags: ["posts", slug], // ⚠️データフェッチ結果をtaggingすることができない
    },
  });
}
```

当然ながら、データフェッチの戻り値をベースにtaggingするような実装はできません。また、Full Route CacheとData Cacheは密結合な関係にあり、影響範囲は非常に複雑でした。

## 設計・プラクティス

Cache Componentsでは`"use cache"`で関数やコンポーネントをCacheするため、従来よりも影響範囲が明確で、RSCとの親和性が高い設計となっています。

RSCでは`"use client"`と`"use server"`2つのディレクティブが導入されましたが、これらは[「2つの世界、2つのドア」](part_2_bundle_boundary#%E3%80%8C2%E3%81%A4%E3%81%AE%E4%B8%96%E7%95%8C%E3%80%812%E3%81%A4%E3%81%AE%E3%83%89%E3%82%A2%E3%80%8D)と考えることができます。Next.jsの`"use cache"`はRSCの世界を拡張し、もう1つ**Cachedな世界とドア**を導入したものと考えることができます。

![RSCの世界観とNext.jsのCache](/images/nextjs-caching-history/cache-door.png)
_RSCの世界観とNext.jsのCache_

### `"use cache"`のユースケース

`"use cache"`は、Static Shellに動的処理やコンポーネントを含めることを主なユースケースとして想定していますが、Dynamic Content内から参照することも可能で、この場合にはインメモリCache^[次章の[`cacheHandlers.default`](cache_stores_setting#use-cache-default)を設定することで、保存先を変更することができます。]のマーカーとして機能します。

しかし、現代のホスティング環境は多くの場合複数プロセスで動作するため、インメモリなCacheはリクエスト間で共有できません。リクエスト間でCacheを共有したい場合には、`"use cache"`ではなく、次章で解説する[`"use cache: remote"`](cache_stores_setting)を利用しましょう。

### `cacheLife(profile)`

[`cacheLife(profile)`↗︎](https://nextjs.org/docs/app/api-reference/functions/cacheLife)は、Cacheの有効期限を設定するための関数です。`profile`はCacheの有効期限に関する以下3つのタイミングを指定することができ、これらをまとめた設定であるprofile文字列やObjectを渡すことができます。

- `stale`: クライアントサイドでキャッシュを利用する期間
- `revalidate`: 次のリクエスト時にバックグラウンドでデータが更新されるまでの期間
- `expire`: 次のリクエスト時にレンダリングをブロックしてデータを再取得するまでの期間

```ts
// 'days': stale: 5 minutes, revalidate: 1 day, expire: 1 week
cacheLife("days");

// 5 minutes
cacheLife({ stale: 300 });
```

Next.jsが提供する`profile`は以下のようになっています。

| プロファイル | 主な用途                             | stale | revalidate | expire |
| ------------ | ------------------------------------ | ----- | ---------- | ------ |
| `default`    | 標準的なコンテンツ                   | 5分   | 15分       | 1年    |
| `seconds`    | リアルタイムデータ                   | 30秒  | 1秒        | 1分    |
| `minutes`    | 頻繁に更新されるコンテンツ           | 5分   | 1分        | 1時間  |
| `hours`      | 1日に複数回更新されるコンテンツ      | 5分   | 1時間      | 1日    |
| `days`       | 毎日更新されるコンテンツ             | 5分   | 1日        | 1週間  |
| `weeks`      | 毎週更新されるコンテンツ             | 5分   | 1週間      | 30日   |
| `max`        | ほとんど変化しない安定したコンテンツ | 5分   | 30日       | 1年    |

### `cacheTag(...tags)`

[`cacheTag(...tags)`↗︎](https://nextjs.org/docs/app/api-reference/functions/cacheTag)は、CacheにtaggingするためのAPIです。`updateTag()`や`revalidateTag()`で指定したtagが付与されたCacheを更新することができます。

```tsx
import { cacheTag } from "next/cache";

export async function getPosts() {
  "use cache";

  cacheTag("posts");

  return await fetch("/api/posts").then((res) => res.json());
}
```

### `updateTag(tag)`, `revalidateTag(tag)`

[`updateTag(tag)`↗︎](https://nextjs.org/docs/app/api-reference/functions/updateTag)は、対象のtagに関連するCacheを**即座に更新**するためのAPIです。更新された結果は、即座に画面に反映されます。

一方、[`revalidateTag(tag)`↗︎](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)も同様にCacheを更新するためのAPIですが、こちらはバックグラウンドで更新を行い、その間は古いCacheを参照します。

```tsx
"use server";

import { updateTag } from "next/cache";

export default async function updatePost() {
  // ...

  updateTag("my-data"); // or `revalidateTag("my-data");`
}
```

## トレードオフ

### 自動で算出されるCacheのkey

`"use cache"`によるCacheのkeyは、Next.jsによって[自動で算出↗︎](https://nextjs.org/docs/app/api-reference/directives/use-cache#cache-keys)されます。

1. Build ID: 各ビルドごとに一意なID
2. Function ID: コード内の関数の場所やシグネチャから計算される安全なハッシュ
3. Serializable arguments: シリアライズ可能なコンポーネントのPropsや関数の引数
4. HMR refresh hash: ホットモジュールリプレース時に生成されるハッシュ（開発時のみ）

特に留意すべき点として、コンポーネントのPropsや関数の引数をキーとして使用しますが、`ReactNode`などシリアライズできないものは**キーに含まれません**。

このような仕組みのため、`"use cache"`においても[Compositionパターン](part_2_composition_pattern)が適用できます。

```tsx
export default function Page() {
  // <CachedComponents>はCacheされる
  // <NotCachedComponent>はchildrenなのでCacheされない
  return (
    <CachedComponents>
      <NotCachedComponent />
    </CachedComponents>
  );
}
```
