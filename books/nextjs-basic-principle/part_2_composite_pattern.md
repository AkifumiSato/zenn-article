---
title: "Client Boundaryとcomposition"
---

## 要約

compositionパターンを駆使して、Client Boundaryを制御しましょう。

## 背景

App RouterではデフォルトがServer Componentsとなっていますが、`"use client";`を記述すればClient Componentsにすることが可能です。当然のことながらClient Componentsはクライアントサイドでも実行されるので、JavaScriptバンドルサイズは増加します。このため、不用意にコンポーネントをClient Componentsにすべきではありません。

一方、React Server ComponentsにおいてServer Components/Client Componentsの依存関係には実装上のルールが存在します。コンポーネントツリーを設計する際には、これらのルールも加味した上で設計を行う必要があります。

### Client ComponentsはServer Componentsを`import`できない

最も重要なルールの1つはClient Componentsは**Server Componentsを`import`できない**ということです。そのため、以下のような実装はできません。

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

しかしこれは依存関係におけるルールであって、Reactツリー上ではClient Components配下にServer Componentsをレンダリングすることがサポートされています。具体的には、**propsでServer Componentsを渡すこと**でServer Componentsをレンダリングすることが可能です。

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

このような実装パターンはいわゆるデザインパターンの一種で、**compositionパターン**と呼ばれています。

### Client Boundary

さて、もう1つの重要なルールは`"use client";`が記述されたファイル以降のモジュール依存関係ツリーは**全てClient Componentsとして扱われる**ということです。

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

上記の`<CompanyLinks>`は`"use client";`が記述されていません。このコンポーネントはClient Componentsでしょうか、Server Componentsでしょうか？

答えは**どちらにもなり得る**です。`<CompanyLinks>`は`"use client";`が記述されたファイル内で`import`された場合にはClient Componentsとして扱わ、Server Componentsから`import`される場合はServer Componentsとして扱われます。

このように、`"use client":`は依存関係ツリーにおいて境界(Boundary)を定義するものであり、この境界はよく**Client Boundary**と表現されます。

## 設計・プラクティス

React Server ComponentsにおいてはClient Boundaryを制御し、不用意にClient Componentsになってしまうことのないように設計する必要があります。Client Boundaryを制御する方法は大きく以下2つです。

### Client ComponentsをReact treeの下層に移動する

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

上記の方法でどうしても上層で`useState`や`useEffect`などが必要になるケースもあります。この場合には**compositionパターン**を活用して、Client Componentsを分離することが有効です。以下はClient Componentsな`<Accordion>`の中にコメント一覧を表示する実装例です。

```tsx
import { Accordion } from "./accordion"; // Client Components
import { fetchComments } from "./fetcher";

export async function Comments() {
  const comments = await fetchComments();

  return (
    <Accordion>
      {comments.map((comment) => (
        <Comment key={comment.id} comment={comment} />
      ))}
    </Accordion>
  );
}
```

## トレードオフ

### RSC Payload転送量の増加

Client ComponentsにするとJavaScriptバンドルサイズが増加します。一方、Server Componentsではページ遷移時などに取得するRSC Payloadの転送量が増加するため、これらはトレードオフの関係にあります。

`useState()`などのclient hooksを使う場合にはClient Componentsにせざるを得ませんが、Client ComponentsにもServer Componentsにもできるようなコンポーネントやツリーについては上記のトレードオフに基づいて最適な判断を行う必要があります。

例えば以下の例において、`<ProductDetail>`は非同期処理などを含まず大きなReactツリーを返すのみです。これをServer Componentsとして扱ってしまうと、大部分が静的な記述にも関わらずRSC Payloadとして転送されるため、転送量が大きくなってしまいます。

```tsx
export function ProductDetail({ product }: { product: Product }) {
  return <>{/* `product`を参照するとても大きなDOMツリー */}</>;
}
```

一方`<ProductDetail>`をClient Componentsにすると、RSC Payloadで転送されるのはpropsで渡してる`product`の部分のみになります。このように、RSC Payloadの転送量を最適化したい場合などにもClient Componentsにすることが手段として有効な場合があります。
