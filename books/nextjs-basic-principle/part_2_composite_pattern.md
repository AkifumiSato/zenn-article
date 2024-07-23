---
title: "Client Boundaryとcomposition"
---

## 要約

compositionパターンを駆使して、Client Boundaryを制御しましょう。

## 背景

[第1部](part_1)ではデータフェッチを中心にServer Componentsの設計パターンについて解説してきました。本章ではServer ComponentsにClient Componentsをどう組み合わせて設計していくべきか解説します。

当然ながらClient Componentsはクライアントサイドでも実行されるので、Client Componentsとして扱うファイルが多いほどJavaScriptバンドルサイズは増加します。一方Server Componentsは[RSC Payload](https://nextjs.org/docs/app/building-your-application/rendering/server-components#how-are-server-components-rendered)として転送されるため、Server Componentsとしてレンダリングされるツリーが多いほど転送量が多くなります。これらはトレードオフの関係にあるため、Client Componentsは少なければ少ない方がいいと言うものではありません。クライアントサイドでの実行必要有無以外にも、JavaScriptバンドルサイズとRSC Payload転送量のトレードオフも加味してClient Componentsにする・しないを判断していく必要があります。重要なのは**Client Componentsにする範囲をコントロールできる**ことです。

Client Componentsにする範囲をコントロールするには、特に以下の2つに注意する必要があります。

### ClientはServerを`import`できない

1つはClient ComponentsはServer Componentsを`import`できないという制約です。そのため、以下のような実装はできません。

```tsx
"use client";

import { useState } from "react";
import { UserInfo } from "./user-info"; // Server Components

export function SideMenu() {
  const [open, setOpen] = useState(false);

  return (
    <>
      <UserInfo />
      <div>
        <button type="button" onClick={() => setOpen((prev) => !prev)}>
          toggle
        </button>
        <div>...</div>
      </div>
    </>
  );
}
```

Client Componentsはクライアントでもサーバーサイドでも実行されます。そのため、Client ComponentsがServer Componentsはじめサーバーサイドモジュールに依存できないのは考えてみると当然のことです。

### Client Boundary

もう1つ注意すべきなのは、`"use client";`が記述されたファイル以降のモジュール依存関係ツリーは**全てClient Componentsとして扱われる**ということです。Client Componentsはサーバーモジュールを`import`できない以上、すべての依存関係をクライアントモジュールとして扱います。

`"use client":`はこのように依存関係において境界(Boundary)を定義するものであり、この境界はよく**Client Boundary**と表現されます。

:::message
以下のようにサーバーモジュールに依存せず、`"use client";`もないファイルで`export`されるコンポーネントは、Client Componentsから`import`されればClient Componentsとして扱わ、Server Componentsから`import`されればServer Componentsとして扱われます。

```tsx
export function CompanyLinks() {
  return (
    <ul>
      <li>
        <a href="/about">About</a>
      </li>
      <li>
        <a href="/contact">Contact</a>
      </li>
    </ul>
  );
}
```

:::

## 設計・プラクティス

React Server ComponentsにおいてはClient Boundaryを制御することが重要で、それには大きく以下2つの方法があります。

### Reactツリーの下層に移動する

1つは**Client ComponentsをReact treeの下層に移動する**というシンプルな方法です。例えば検索バーを持つヘッダーを実装する際に、ヘッダーごとClient Componentsにするのではなく検索バーの部分だけClient Componentsとして切り出し、ヘッダー自体はServer Componentsに保つといった方法です。

```tsx
// header.tsx(Server Components)
import { SearchBar } from "./search-bar"; // Client Components

export function Header() {
  return (
    <header>
      <h1>My App</h1>
      <SearchBar />
    </header>
  );
}
```

### compositionパターンを活用する

上記の方法はシンプルな解決策ですが、どうしても上位のコンポーネントで`useState`や`useEffect`などが必要になるケースもあります。この場合には**compositionパターン**を活用して、Client Componentsを分離することが有効です。

前述の通り、Client ComponentsはServer Componentsを`import`することができません。しかしこれは依存関係上の制約であって、ReactツリーとしてはClient Componentsの`children`などのpropsにServer Componentsを渡すことで、レンダリングが可能です。

```tsx
"use client";

import { useState } from "react";

// `children`に`<UserInfo>`などのServer Componentsを渡すことが可能！
export function SideMenu({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);

  return (
    <>
      {children}
      <div>
        <button type="button" onClick={() => setOpen((prev) => !prev)}>
          toggle
        </button>
        <div>...</div>
      </div>
    </>
  );
}
```

このような実装パターンはいわゆるデザインパターンの一種で、compositionパターンと呼ばれています。

## トレードオフ

### RSC Payload転送量を減らすためのClient Components

前述の通り、Server ComponentsはRSC Payloadの転送量が増加するため、Client ComponentsのJavaScriptバンドルサイズの増加とはトレードオフの関係にあります。そのため`useState()`などのhooksを使わない場合においても、転送量を削減する目的でClient Componentsにすることが望ましいケースがあります。

例えば以下の`<Product>`について考えてみます。

```tsx
export async function Product() {
  const product = await fetchProduct();

  return <>{/* `product`を参照するとても大きなReactツリー */}</>;
}
```

hooksなども特になく、ただ`product`を取得・参照して大きなReactツリーを作るだけのコンポーネントです。しかしこの参照してる部分が大きいと、転送コストが大きくなりパフォーマンスが劣化する可能性があります。このようなケースにおいては何度も膨大なRSC Payloadを転送するのではなく、データフェッチ層としてのServer Componentsとそれを受け取るClient Componentsに分離することが有効です。

```tsx
export async function Product() {
  const product = await fetchProduct();

  return <ProductPresentaional product={product} />;
}
```

```tsx
"use client";

export function ProductPresentaional({ product }: { product: Product }) {
  return <>{/* `product`を参照するとても大きなDOMツリー */}</>;
}
```
