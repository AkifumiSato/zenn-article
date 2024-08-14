---
title: "Request Memoization"
---

## 要約

データフェッチ層を分離して、Request Memoizationを生かせる設計を心がけましょう。

## 背景

[_データフェッチ コロケーション_](part_1_colocation)の章で述べた通り、App Routerではデータフェッチをコロケーションすることが推奨されています。しかし末端のコンポーネントでデータフェッチを行うと、ページ全体を通して重複するリクエストが発生する可能性が高まります。App Routerはこれに対処するため、レンダリング中の同一リクエストをメモ化し排除する[Request Memoization](https://nextjs.org/docs/app/building-your-application/caching#request-memoization)を実装しています。

しかし、このRequest Memoizationがリクエストを重複と判定するには、同一URL・同一オプションの指定が必要で、オプションが1つでも異なれば別リクエストが発生してしまいます。

## 設計・プラクティス

オプションの指定ミスによりRequest Memoizationが効かないことなどがないよう、複数のコンポーネントで利用しうるデータフェッチ処理は**データフェッチ層**として分離しましょう。

```ts
// プロダクト情報取得のデータフェッチ層
export async function getProduct(id: string) {
  const res = await fetch(`https://dummyjson.com/products/${id}`, {
    // 独自ヘッダーなど
  });
  return res.json();
}
```

### ファイル構成

上記のようなデータフェッチ層を分離する際のファイル構成には、様々なパターンが考えられます。ここでは上記関数を`/products`配下でのみ利用すると仮定して、筆者が考えるファイル構成例を示します。

- `app/products/api.ts`
- `app/products/_lib/api.ts`
- `app/products/_lib/api/product.ts`
- `app/products/fetcher.ts`
- `app/products/_lib/fetcher.ts`
- `app/products/_lib/fetcher/product.ts`

`/products`配下でのみ利用するならファイルコロケーションするのがApp Router流ですが、ファイルの命名やディレクトリについては開発規模や流儀によって異なるので、自分たちのチームでルールを決めておきましょう。

### `server-only` package

[_データフェッチ on Server Components_](part_1_server_components)で述べたとおり、データフェッチは基本的にServer Componentsで行うことが推奨されます。データフェッチ層を誤ってクライアントサイドで利用することを防ぐためにも、[server-only](https://www.npmjs.com/package/server-only)パッケージを利用することを検討しましょう。

```ts
// Client Compomnentsでimportするとerror
import "server-only";

export async function getProduct(id: string) {
  const res = await fetch(`https://dummyjson.com/products/${id}`, {
    // 独自ヘッダーなど
  });
  return res.json();
}
```

## トレードオフ

特になし
