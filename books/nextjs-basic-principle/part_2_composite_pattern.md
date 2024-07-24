---
title: "Compositionパターン"
---

## 要約

Compositionパターンを駆使して、Client Boundaryを制御しましょう。

## 背景

[第1部](part_1)ではデータフェッチ観点を中心にServer Componentsの設計パターンについて解説してきました。そこにClient Componentsをどう組み合わせていくかを考えることが、React Server Componentsにおけるコンポーネント設計の基礎となります。

Client Componentsは当然ながらクライアントサイドでも実行されるので、Client Componentsが多いほどJavaScriptバンドルサイズは増加します。一方Server Componentsは[RSC Payload](https://nextjs.org/docs/app/building-your-application/rendering/server-components#how-are-server-components-rendered)として転送されるため、レンダリングされるReact treeが多いほど転送量が多くなります。これらはトレードオフの関係にあるため、**Client Componentsは少なければ少ない方がいいと言うものではありません**。

クライアントサイドでの実行必要有無以外にも、JavaScriptバンドルサイズとRSC Payload転送量のトレードオフも加味してClient Componentsにする・しないを判断していく必要があります。重要なのは**Client Componentsにする範囲をコントロールできる**ことです。

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

もう1つ注意すべきなのは、`"use client";`が記述されたファイル以降のモジュール依存関係treeツリーは**全てClient Componentsとして扱われる**ということです。Client Componentsはサーバーモジュールを`import`できない以上、すべての依存関係をクライアントモジュールとして扱います。

`"use client":`はこのように依存関係において境界(Boundary)を定義するものであり、この境界はよく**Client Boundary**と表現されます。

以下のようにサーバーモジュールに依存せず、`"use client";`もないファイルで`export`されるコンポーネントは、Client Componentsから`import`されればClient Componentsとして扱わ、Server Componentsから`import`されればServer Componentsとして扱われます。

```tsx
// `"use client";`がないファイル
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

このようなコンポーネントは、一部で**Shared Components**と表現されることがあります。

## 設計・プラクティス

React Server ComponentsにおいてはClient Boundaryを制御すること、特にClient Boundaryを限定的にするテクニックが重要です。限定的にできるということは必要に応じて範囲を広げることも容易であることと同義です。

これには大きく以下2つの方法があります。

### React treeの下層に移動する

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

こうすることで必要に応じてClient Boundaryに含まれる範囲を増減させることができます。

### Compositionパターンを活用する

上記の方法はシンプルな解決策ですが、どうしても上位のコンポーネントで`useState`や`useEffect`などが必要になるケースもあります。この場合には**Compositionパターン**を活用して、Client Componentsを分離することが有効です。

前述の通り、Client ComponentsはServer Componentsを`import`することができません。しかしこれは依存関係上の制約であって、React treeとしてはClient Componentsの`children`などのpropsにServer Componentsを渡すことで、レンダリングが可能です。

前述の`<SideMenu>`の例を書き換えてみます。

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

Server ComponentsもClient Componentsも等しく`React.ReactNode`として表現され、tree構造を表現できています。これがいわゆるCompositionパターンと呼ばれる実装パターンです。

## トレードオフ

### RSC Payload転送量を減らすためのClient Components

前述の通りServer ComponentsはRSC Payloadの転送量が増加するため、Client ComponentsのJavaScriptバンドルサイズの増加とはトレードオフの関係にあります。そのため`useState()`などのhooksを使わない場合においても、RSC Payloadの転送量を削減する目的でClient Componentsにすることが望ましいケースがあります。

例えば以下の`<Product>`について考えてみます。

```tsx
export async function Product() {
  const product = await fetchProduct();

  return <>{/* `product`を参照するとても大きなReact tree */}</>;
}
```

hooksなども特になく、ただ`product`を取得・参照して大きなReact treeを作るだけのコンポーネントです。しかしこのデータを参照してるReact tree部分が大きいと、RSC Payloadの転送コストが大きくなりパフォーマンス劣化を引き起こす可能性があります。特に低速なネットワーク環境においてページ遷移を繰り返す際などには影響が顕著になりがちです。

このようなケースにおいてはServer Componentsでデータフェッチのみを行い、React tree部分はClient Componentsに分離することでRSC Payloadの転送量を削減することができます。

```tsx
export async function Product() {
  const product = await fetchProduct();

  return <ProductPresentaional product={product} />;
}
```

```tsx
"use client";

export function ProductPresentaional({ product }: { product: Product }) {
  return <>{/* `product`を参照するとても大きなReact tree */}</>;
}
```
