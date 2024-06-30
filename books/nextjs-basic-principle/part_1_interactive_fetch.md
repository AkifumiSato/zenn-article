---
title: "ユーザー操作とデータフェッチ"
---

## 背景

## 設計・プラクティス

## 結論

## 例外

## memo

### `useActionState`とServer Actionsを組み合わせて使う場合

ユーザーの入力に基づいてデータ取得を行いたいケースにおいては、[Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)と[useActionState](https://react.dev/reference/react/useActionState)(旧: `useFormState`)を利用して実装することが可能です。

以下はユーザーの入力に基づいて商品を検索する実装例です。

```ts
// app/actions.ts
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

```tsx
// app/form.tsx
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

検索したい文字列を入力し、Submitボタンを押すとヒットした商品の名前が表出するようになっています。単純なページであれば、以下公式チュートリアルの実装例のように`router.replace()`によってURLを更新・Server Componentsごと再実行する手段もあります。

https://nextjs.org/learn/dashboard-app/adding-search-and-pagination

しかしURLを変更したくない・他のコンテンツ部分を再レンダリングしたくないなどの場合、このようにServer Actionsと`useActionState`を利用することを検討すると良いでしょう。
