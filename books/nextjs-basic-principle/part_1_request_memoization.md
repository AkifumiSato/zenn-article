---
title: "Request Memoization"
---

## 要約

データフェッチ層を分離して、Request Memoizationを生かせる設計を心がけましょう。

## 背景

[データフェッチ コロケーション](part_1_colocation)の章で述べた通り、Next.jsではデータフェッチをコロケーションすることが推奨されています。末端のコンポーネントでデータフェッチを行うことはリクエストの重複リスクを伴いますが、Next.jsでは[**Request Memoization**↗︎](https://nextjs.org/docs/app/guides/caching#request-memoization)（リクエストのメモ化）によってレンダリング中の同一リクエストを排除します。

しかし、Next.jsがリクエストを重複と判定するには、同一URL・同一オプションの指定が必要で、オプションが1つでも異なれば別リクエストが発生してしまいます。

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

Next.jsではコロケーションを強く意識した設計がなされているので、データフェッチ層をファイル分離する場合にもファイルコロケーションすることが推奨されます。

前述の`getProduct()`を分離する場合、筆者なら以下のいずれかのような形でファイルを分離します。データフェッチ層が多い場合にはより細かく分離すると良いでしょう。

| ファイル                               | 補足                                                                |
| -------------------------------------- | ------------------------------------------------------------------- |
| `app/products/fetcher.ts`              | `products/`以下にファイルが少ない<br>データフェッチ関数も少ない場合 |
| `app/products/_lib/fetcher.ts`         | `products/`以下にファイルが多い<br>データフェッチ関数は少ない場合   |
| `app/products/_lib/fetcher/product.ts` | データフェッチ関数が多い場合                                        |

ファイルの命名やディレクトリについては開発規模や流儀によって異なるので、自分たちのチームでルールを決めておきましょう。

### `server-only`

[データフェッチ on Server Components](part_1_server_components)で述べたとおり、データフェッチは基本的にServer Componentsで行うことが推奨されます。データフェッチ層を誤ってクライアントサイドで利用することを防ぐためにも、[server-only↗︎](https://www.npmjs.com/package/server-only)を利用してモジュールを保護することを推奨します。

```ts
// Client Bundle内でimportするとエラー
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
