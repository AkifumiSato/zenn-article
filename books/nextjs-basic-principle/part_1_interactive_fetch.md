---
title: "ユーザー操作とデータフェッチ"
---

## 要約

ユーザー操作に基づくデータフェッチと再レンダリングには、Server Actionsと`useActionState()`を利用しましょう。

## 背景

[_データフェッチ on Server Components_](part_1_server_components)で述べた通り、App RouterにおいてデータフェッチはServer Componentsで行うことが基本形です。しかし、ユーザー操作に基づいてデータフェッチ・再レンダリングを行うのにServer Componentsは適していません。App Routerにおいては`router.refresh()`などでページ全体を再レンダリングすることはできますが、ユーザー操作に基づいて部分的に再レンダリングしたい場合には不適切です。

## 設計・プラクティス

App RouterがサポートしてるReact Server Componentsにおいては、[Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)と`useActionState()`(旧: `useFormState()`)を利用することで、ユーザー操作に基づいたデータフェッチを実現できます。

### `useActionState()`

`useActionState()`は関数と初期値を渡すことで、Server Actionsによってのみ更新できるState管理が実現できます。

https://react.dev/reference/react/useActionState

以下はユーザーの入力に基づいて商品を検索する実装例です。Server Actionsとして定義された`searchProducts()`を`useActionState()`の第一引数に渡しており、formがサブミットされるごとに実行されます。

```ts :app/actions.ts
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
import { searchProducts } from "./actions";

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

これにより、 検索したい文字列を入力・サブミットすると、検索ヒットした商品の名前が表出するようになっています。

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
export default async function Page({
  searchParams,
}: {
  searchParams?: {
    query?: string;
    page?: string;
  };
}) {
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

この場合、Server Actionsと`useActionState()`のみでは実現できないリロード復元やURLシェアが実現できます。上記例のように検索が主であるページにおいては、状態をURLに保存することを検討すべきでしょう。`useActionState()`を使いつつ、状態をURLに保存することもできます。

一方サイドナビゲーションやcmd+kで開く検索モーダルのように、リロード復元やURLシェアをする必要がないケースでは、Server Actionsと`useActionState()`の実装が非常に役立つことでしょう。

### データ操作に伴う再レンダリング

ここで紹介したのはユーザー操作に伴うデータフェッチ、つまり**データ操作を伴わない**場合の設計パターンです。ユーザー操作にともなってデータを操作し、その後の結果を再取得したいこともあります。これはServer Actionsと`revalidatePath()`/`revalidateTag()`を組み合わせ実行することで実現できます。

これについては、後述の[_データ操作とServer Actions_](part_3_data_mutation)にて詳細を解説します。
