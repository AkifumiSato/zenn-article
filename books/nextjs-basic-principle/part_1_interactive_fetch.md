---
title: "ユーザー操作とデータフェッチ"
---

## 要約

ユーザー操作に基づくデータフェッチと再レンダリングには、Server Functionsと`useActionState()`を利用しましょう。

## 背景

[データフェッチ on Server Components](part_1_server_components)で述べた通り、Next.jsにおいてデータフェッチはServer Componentsで行うことが基本形です。しかし、Server Componentsはユーザー操作に基づいてデータフェッチ・再レンダリングを行うのに適していません。Next.jsでは`router.refresh()`などでページ全体を再レンダリングすることはできますが、部分的に再レンダリングしたい場合には不適切です。

## 設計・プラクティス

Next.jsがサポートしてるRSCにおいては、[Server Functions↗︎](https://ja.react.dev/reference/rsc/server-functions)と`useActionState()`を利用することで、ユーザー操作に基づいたデータフェッチを実現できます。

### `useActionState()`

`useActionState()`は関数と初期値を渡すことで、Server Functionsを通して更新できるState管理が実現できます。

https://ja.react.dev/reference/react/useActionState

以下はユーザーの入力に基づいて商品を検索する実装例です。Server Functionsとして定義された`searchProducts()`を`useActionState()`の第一引数に渡しており、formがサブミットされるごとに`searchProducts()`が実行されます。

```ts :app/search-products.ts
"use server";

export async function searchProducts(
  _prevState: Product[],
  formData: FormData,
) {
  const query = formData.get("query") as string;
  const res = await fetch(`https://dummyjson.com/products/search?q=${query}`);
  const { products } = (await res.json()) as { products: Product[] };

  return products;
}

// ...
```

```tsx :app/form.tsx
"use client";

import { useActionState } from "react-dom";
import { searchProducts } from "./search-products";

export default function Form() {
  const [products, action] = useActionState(searchProducts, []);

  return (
    <>
      <form action={action}>
        <label htmlFor="query">
          Search Product:&nbsp;
          <input type="text" id="query" name="query" />
        </label>
        <button type="submit">Submit</button>
      </form>
      <ul>
        {products.map((product) => (
          <li key={product.id}>{product.title}</li>
        ))}
      </ul>
    </>
  );
}
```

上記実装例では検索したい文字列を入力しサブミットすると、検索ヒットした商品の名前が一覧で表示されます。

## トレードオフ

### URLシェア・リロード対応

form以外ほとんど要素がないような単純なページであれば、公式チュートリアルの実装例のように`router.replace()`によってURLを更新・ページ全体を再レンダリングするという手段があります。

https://nextjs.org/learn/dashboard-app/adding-search-and-pagination

:::details チュートリアルの実装例(簡易版)

```tsx
"use client";

import { useSearchParams, usePathname, useRouter } from "next/navigation";

export default function Search() {
  const searchParams = useSearchParams();
  const pathname = usePathname();
  const { replace } = useRouter();

  function handleSearch(term: string) {
    // MEMO: 実際にはdebounce処理が必要
    const params = new URLSearchParams(searchParams);
    if (term) {
      params.set("query", term);
    } else {
      params.delete("query");
    }
    replace(`${pathname}?${params.toString()}`);
  }

  return (
    <input
      onChange={(e) => handleSearch(e.target.value)}
      defaultValue={searchParams.get("query")?.toString()}
    />
  );
}
```

```tsx
export default async function Page(props: {
  searchParams?: Promise<{
    query?: string;
    page?: string;
  }>;
}) {
  const searchParams = await props.searchParams;
  const query = searchParams?.query || "";
  const currentPage = Number(searchParams?.page) || 1;

  return (
    <div>
      <Search />
      {/* ... */}
      <Suspense key={query + currentPage} fallback={<InvoicesTableSkeleton />}>
        <Table query={query} currentPage={currentPage} />
      </Suspense>
      {/* ... */}
    </div>
  );
}
```

:::

この場合、Server Functionsと`useActionState()`のみでは実現が難しいリロード復元やURLシェアが実現できます。上記例のように検索が主であるページにおいては、状態をURLに保存すること^[URLに状態を保存するには、[`location-state`↗︎](https://github.com/recruit-tech/location-state)や[`nuqs`↗︎](https://nuqs.dev/)などのライブラリの利用が便利です。]を検討すべきでしょう。`useActionState()`を使いつつ、状態をURLに保存することもできます。

一方サイドナビゲーションやcmd+kで開く検索モーダルのように、リロード復元やURLシェアをする必要がないケースでは、Server Functionsと`useActionState()`の実装が非常に役立つことでしょう。

### データ操作に伴う再レンダリング

ここで紹介したのはユーザー操作に伴うデータフェッチ、つまり**データ操作を伴わない**場合の設計パターンです。しかし、ユーザー操作にともなってデータを操作し、その後の結果を再取得したいこともあります。これはServer Actions^[データ操作を伴うServer Functionsは、**Server Actions**と呼ばれます。[参考↗︎](https://nextjs.org/docs/app/getting-started/updating-data#what-are-server-functions)]と`revalidatePath()`/`revalidateTag()`を組み合わせ実行することで実現できます。

これについては、後述の[データ操作とServer Actions](part_3_data_mutation)にて詳細を解説します。
