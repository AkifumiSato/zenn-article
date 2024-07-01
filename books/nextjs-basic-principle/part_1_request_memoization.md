---
title: "Request Memoization"
---

## 要約

Request Memoizationによってリクエストの重複が排除されることを生かしたデータフェッチ層を設計しましょう。

## 背景

[コロケーション](part_1_colocation)のチャプターで述べた通り、App Routerではデータフェッチをコロケーションすることが推奨されています。しかし末端のコンポーネントでデータフェッチを行うと、ページ全体を通して重複するリクエストが発生する可能性が高まります。App Routerはこれに対処するため、レンダリング中の同一リクエストを排除しメモ化する[Request Memoization](https://nextjs.org/docs/app/building-your-application/caching#request-memoization)を提供しています。

しかしこのRequest Memoizationがリクエストを重複と判定するには、同一URL・同一オプションの指定が必要で、オプションが1つでも異なれば別リクエストになってしまいます。

## 設計・プラクティス

オプションの指定ミスによりRequest Memoizationが効かないことなどないよう、複数のコンポーネントで利用しうるデータフェッチ処理は**データフェッチ層**として分離しましょう。

```ts
// プロダクト情報取得のデータフェッチ層
export async function getProduct(id: string) {
  const res = await fetch(`https://dummyjson.com/products/${id}`);
  return res.json();
}
```

### ファイル構成

上記のようなデータフェッチ層のファイル構成には様々なパターンが考えられます。ここでは上記関数を`/products`配下でのみ利用すると仮定して、筆者が考えるファイル構成例を示します。

- `app/products/api.ts`
- `app/products/_lib/api.ts`
- `app/products/_lib/api/product.ts`
- `app/products/fetcher.ts`
- `app/products/_lib/fetcher.ts`
- `app/products/_lib/fetcher/product.ts`

### `server-only` package

[データフェッチ on Server Components](part_1_server_components)で述べたとおり、データフェッチは基本的にServer Componentsで行うことが推奨されます。データフェッチ層を誤ってクライアントサイドで利用することを防ぐためにも、[server-only](https://www.npmjs.com/package/server-only)パッケージを利用することを検討しましょう。

```ts
import "server-only";

export async function getProduct(id: string) {
  const res = await fetch(`https://dummyjson.com/products/${id}`);
  return res.json();
}
```

### preloadパターン

末端のコンポーネントでのデータフェッチはより上位のコンポーネントの非同期処理に依存する**ウォーターフォール**が発生しがちですが、データフェッチ間にデータの依存関係がないなら並列なデータフェッチが理想です。

![water fall data fetch](/images/nextjs-basic-principle/sequential-fetching.png)

Request Memoizationを利用した[preloadパターン](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns#preloading-data)を利用すれば、並列なデータフェッチを実現できます。

```ts
// app/products/fetcher.ts
import "server-only";

export const preload = (id: string) => {
  void getProduct(id);
};

export async function getProduct(id: string) {
  const res = await fetch(`https://dummyjson.com/products/${id}`);
  return res.json();
}
```

```tsx
// app/products/[id].tsx
export default async function Page({
  params: { id },
}: {
  params: { id: string };
}) {
  preload(id);

  // ...
}
```

## トレードオフ

特になし
