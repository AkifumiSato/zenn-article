---
title: "ユーザー操作とデータフェッチ"
---

## 要約

ユーザー操作に基づくデータフェッチと部分的な更新の実装は、Server Actionsと`useActionState`を利用しましょう。

## 背景

- [データフェッチ on Server Components](part_1_server_components)で述べた通り、RSCにおいてはデータフェッチはServer Componentsで行うことが基本形です
- しかし、App Routerでは`router.refresh()`などでページ全体を再レンダリングすることはできますが、部分的にServer Componentsを再レンダリングするということができません。
- つまり、ユーザー操作に基づいたデータ取得とServer Componentsは相性が悪いということになります。

## 設計・プラクティス

- Reactではユーザー操作に基づいたデータフェッチは、Server ActionsとuseActionStateを利用して実装することが推奨されます
- 基本はServer Components、ユーザー操作に基づく部分的更新ならServer Actionsと[useActionState](https://react.dev/reference/react/useActionState)(旧: `useFormState`)という分け方になります。

### useActionState

- useActionStateはServer Actionsと初期値を必須引数となっています。
- これによりServer Actionsによってのみ更新できるState管理が実現できます。

### 実装例

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

検索したい文字列を入力し、Submitボタンを押すとヒットした商品の名前が表出するようになっています。他にほとんど要素がない単純なページであれば、以下公式チュートリアルの実装例のように`router.replace()`によってURLを更新・Server Componentsを再レンダリングする手段もあります。

https://nextjs.org/learn/dashboard-app/adding-search-and-pagination

しかし、URLを変更したくない・他のコンテンツ部分を再レンダリングしたくないなどの場合、このようにServer Actionsと`useActionState`を利用することを検討すると良いでしょう。

## トレードオフ

前述の[公式チュートリアルの実装例](https://nextjs.org/learn/dashboard-app/adding-search-and-pagination)のようにURLを更新する場合、リロード復元やURLシェアが実現できます。一方Server ActionsとuseActionStateの場合、リロード復元やURLシェアは実現できません。

Google検索のように検索が主要な機能のページにおいては状態をURLに保存することを検討すべきでしょう。一方サイドナビゲーションやcmd+kで開く検索モーダルのようにリロード復元やURLシェアをすべきでないケースもあります。こういった場合にはServer ActionsとuseActionStateが非常に役立つことでしょう。
