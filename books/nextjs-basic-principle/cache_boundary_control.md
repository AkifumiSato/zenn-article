---
title: "Cacheの境界と制御"
---

## 要約

`"use cache"`はComponentや関数がCache可能であることを宣言するもので、Cacheの制御関数と組み合わせて利用します。制御関数の種類や役割を理解して、Cacheを活用しましょう。

## 背景

:::message
以下は筆者の過去記事[Next.js Cache回想↗︎](https://zenn.dev/akfm/articles/nextjs-caching-history)と重複します。より詳細な内容を知りたい方はこちらの記事をご参照ください。
:::

Next.js開発チームは当初、開発者が意識しなくて済むような暗黙的なCache制御こそ理想的だと考えていました。そのため、App Router登場当初のCacheはデフォルトで有効で、`headers()`や`fetch()`のオプションなどによって暗黙的にOpt-outされるような設計になっていました。また、当時はPPRの実装や概念も存在しなかったため、Route単位でStatic/Dynamic Renderingを切り替えるしかなく、多層のCacheによる最適化が試みられていました。

このような**暗黙的で多層のCache戦略**は開発者に多くの混乱を招いたため、Cache Componentsでは従来の戦略から脱却すべく、**明示的で合成可能なCache戦略**を採用しています。

## 設計・プラクティス

`"use cache"`はComponentや関数が**Cache可能であること**を宣言するものです。言い換えると、Next.jsは`"use cache"`が宣言されたComponentや関数を必ずしも**Cacheするとは限りません**。

RSCでは`"use client"`と`"use server"`2つのディレクティブが導入されましたが、これらは[「2つの世界、2つのドア」](part_2_bundle_boundary#%E3%80%8C2%E3%81%A4%E3%81%AE%E4%B8%96%E7%95%8C%E3%80%812%E3%81%A4%E3%81%AE%E3%83%89%E3%82%A2%E3%80%8D)と考えることができます。Next.jsの`"use cache"`はRSCの世界を拡張し、もう1つ**Cachedな世界とドア**を導入したものと考えることができます。

![RSCの世界観とNext.jsのCache](/images/nextjs-caching-history/cache-door.png)
_RSCの世界観とNext.jsのCache_

### `"use cache"`

`"use cache"`は、Static Shellに動的処理やコンポーネントを含めることを主なユースケースとして想定していますが、Dynamic Content内から参照することも可能で、この場合にはインメモリCache^[次章の[`cacheHandlers.default`](cache_stores_setting#use-cache-default)を設定することで、保存先を変更することができます。]として扱われます。

しかし、セルフホスティングでは多くの場合マルチプロセスで動作するため、インメモリなCacheはリクエスト間で共有できない可能性がありますが、リクエスト間でCacheを共有したいという要件は非常に一般的です。リクエスト間でCacheを共有したい場合には、`"use cache"`ではなく、次章で解説する[`"use cache: remote"`](cache_stores_setting)を利用しましょう。

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

Next.jsがデフォルトで提供する`profile`は以下のようになっています。

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

[`updateTag(tag)`↗︎](https://nextjs.org/docs/app/api-reference/functions/updateTag)は、指定したtagが付与されたCacheを**即座に更新**するためのAPIです。更新された結果は、即座に画面に反映されます。

一方、[`revalidateTag(tag)`↗︎](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)も同様にCacheを更新するためのAPIですが、こちらはバックグラウンドで更新を行い、その間は古いCacheを参照します。

```tsx
"use server";

import { updateTag } from "next/cache";

export default async function updatePost() {
  // ...

  updateTag("my-data");
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

この制約により、`"use cache"`においても[Compositionパターン](part_2_composition_pattern)が適用できます。

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
