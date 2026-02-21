---
title: "Static Renderingの境界と制御"
---

## 要約

`"use cache"`はComponentや関数がCache可能であることを宣言するもので、Cacheの制御関数と組み合わせて利用します。Cacheを活用し、高いパフォーマンスや耐障害性の実現を心がけましょう。

## 背景

Next.js開発チームはApp Routerの開発当初、暗黙的なCache制御により、開発者がCacheを意識しなくて済むような世界観こそ理想的だと考えていました。そのため、App Router登場当初のCacheはデフォルトで有効で、`headers()`や`fetch()`のオプションなどによって暗黙的にOpt-outされるような設計になっていました。

しかし、実際にはこのような設計は開発者に多くの混乱を招いたため、慎重な検討を重ねながら改善を進めていきました。以下はその改善の一部です。

- ドキュメントの改善
- `fetch()`のデフォルトCache廃止
- `staleTimes`オプションの追加

しかし、根本的な設計思想の改善はリアーキテクチャなしには困難でした。

Cache Componentsは新しい世界観として再設計されたもので、Cacheは`"use cache"`によってOpt-inできるような設計に変更されました。

:::message
より詳しい歴史的経緯については、筆者の過去の記事[Next.js Cache回想↗︎](https://zenn.dev/akfm/articles/nextjs-caching-history)をご参照ください。
:::

## 設計・プラクティス

`"use cache"`はComponentや関数が**Cache可能であること**を宣言するものです。言い換えると、Next.jsは`"use cache"`が宣言されたComponentや関数を必ずしも**Cacheするとは限りません**。

### `"use cache"`

`"use cache"`は、Static Shellに動的処理やコンポーネントを含めることを主なユースケースとして想定してますが、Dynamic Content内の一部から利用することも可能で、この場合にはインメモリCache^[次章の[`cacheHandlers.default`](./cache_stores_setting#use-cache-default)を設定することで、保存先を変更することができます。]として扱われます。

しかし、セルフホスティングでは多くの場合マルチプロセスで動作するため、インメモリなCacheはリクエスト間で共有できない可能性がありますが、リクエスト間でCacheを共有したいという要件は非常に一般的です。リクエスト間でCacheを共有したい場合には、`"use cache"`ではなく、[次章で解説する`"use cache: remote"`](./cache_stores_setting)を利用しましょう。

:::message
Vercelはサーバーレス環境で動作するため、`"use cache"`ではCacheされないケースがあるそうです。詳細はこちらの[issueのコメント↗︎](https://github.com/vercel/next.js/issues/85240#issuecomment-3560124078)をご参照ください。
:::

### `cacheLife(profile)`

[`cacheLife(profile)`↗︎](https://nextjs.org/docs/app/api-reference/functions/cacheLife)は、Cacheの有効期限を設定するための関数です。`profile`はCacheの有効期限に関する以下3つのタイミングを指定することができ、これらをまとめた設定であるprofile文字列やObjectを渡すことができます。

- `stale`: この時間内は、クライアントがサーバーをチェックせずにキャッシュされたデータを使用します
- `revalidate`: この時間が経過すると、次のリクエスト時にバックグラウンドでデータが更新されます
- `expire`: この時間が経過すると、次のリクエスト時にデータを更新します

```ts
// 'days': stale: 5 minutes, revalidate: 1 day, expire: 1 week
cacheLife("days");

// 5 minutes
cacheLife({ stale: 300 });
```

Next.jsが提供する`profile`は以下のようになっています。カスタムプロファイルを設定することや、`profile`にObjectで詳細な設定を渡すことも可能です。

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

[`updateTag(tag)`↗︎](https://nextjs.org/docs/app/api-reference/functions/updateTag)は、指定したtagが付与されたCacheを**更新**するためのAPIです。[`revalidateTag(tag)`↗︎](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)も同様にCacheを更新するためのAPIですが、こちらはバックグラウンドで更新を行い、その間は古いCacheを参照します。

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

`"use cache"`によるCacheのkeyは、Next.jsによって自動で算出されます。

1. Build ID: 各ビルドごとに一意なID
2. Function ID: コード内の関数の場所やシグネチャから計算される安全なハッシュ
3. Serializable arguments: コンポーネントのPropsや関数の引数（シリアライズ可能なもの）
4. HMR refresh hash（開発時のみ）: ホットモジュールリプレース時にキャッシュを無効化します

特に3.のSerializable argumentsは、コンポーネントのPropsや関数の引数をキーとして使用しますが、`ReactNode`などシリアライズできないものは**キーに含まれません**。この仕組みにより、`"use cache"`を使用したコンポーネントはCacheのkeyに`children`を含みません。

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
