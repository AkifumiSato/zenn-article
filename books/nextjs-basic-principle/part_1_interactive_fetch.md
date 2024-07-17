---
title: "ユーザー操作とデータフェッチ"
---

## 要約

ユーザー操作に基づくデータフェッチと再レンダリングには、Server Actionsと`useActionState`を利用しましょう。

## 背景

[データフェッチ on Server Components](part_1_server_components)で述べた通り、RSCにおいてデータフェッチはServer Componentsで行うことが基本形です。しかし、ユーザー操作に基づいてデータフェッチ・再レンダリングを行うのにServer Componentsは適していません。App Routerにおいては`router.refresh()`などでページ全体を再レンダリングすることはできますが、ユーザー操作に基づいて部分的に再レンダリングしたい場合には不適切です。

## 設計・プラクティス

Reactではユーザー操作に基づいたデータフェッチを実現するために、[Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)と[useActionState](https://react.dev/reference/react/useActionState)(旧: `useFormState`)などのhooksが提供されています。

データ取得の基本系はServer Components、ユーザー操作に基づく部分的更新ならServer Actionsと`useActionState`という使い分けになります。

### `useActionState`

`useActionState`はServer Actionsと初期値が必須引数となっており、Server Actionsによってのみ更新できるState管理が実現できます。

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

検索したい文字列を入力し、Submitボタンを押すとヒットした商品の名前が表出するようになっています。

## トレードオフ

### URLシェア・リロード対応

上記実装例の`<Form>`以外ほとんど要素がないような単純なページであれば、公式チュートリアルの実装例のように`router.replace()`によってURLを更新・ページ全体を再レンダリングするという手段があります。

https://nextjs.org/learn/dashboard-app/adding-search-and-pagination

この場合、Server Actionsと`useActionState`では実現できないリロード復元やURLシェアが実現できます。

上記のような検索が最も主であるページにおいては状態をURLに保存することを検討すべきでしょう。一方サイドナビゲーションやcmd+kで開く検索モーダルのようにリロード復元やURLシェアをすべきでないケースでは、Server Actionsと`useActionState`の実装が非常に役立つことでしょう。

### データ操作に伴う再レンダリング

ここで紹介したのはユーザー操作に伴うデータフェッチ、つまりデータ操作を伴わない場合の設計パターンです。ユーザー操作にともなってデータ操作・操作後の結果を再取得したいこともあります。これはServer Actionsと`revalidatePath`/`revalidateTag`を組み合わせ実行することで実現できます。

これについては、後述の[データ操作とServer Actions](part_2_data_mutation_inner)にて詳細を解説します。
