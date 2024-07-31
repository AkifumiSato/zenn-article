---
title: "Compositionパターン"
---

## 要約

Compositionパターンを駆使してServer Components中心に組み立てたコンポーネントツリーから、Client Componentsを適切に切り分けましょう。

## 背景

[第1部](part_1)で述べたように、React Server Componentsのメリットを活かすにはServer Components中心の設計が重要となります。そのため**Client Componentsは適切に分離・独立**していることが好ましいですが、そのためにはClient Componentsにおける依存関係の2つの制約について考慮する必要があります。

### Client Componentsはサーバーモジュールを`import`できない

1つはClient ComponentsはServer Componentsはじめサーバーモジュールを`import`できないという制約です。クライアントサイドでも実行される以上、サーバーモジュールに依存できないのは考えてみると当然のことです。

そのため、以下のような実装はできません。

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

:::message
この制約に対し唯一例外となるのが`"use server";`が付与されたファイルや関数、つまりServer Actionsです。
:::

### Client Boundary

もう1つ注意すべきなのは、`"use client";`が記述されたモジュールから`import`されるモジュール以降は全て暗黙的にクライアントモジュールとして扱われ、それらで定義されたコンポーネントは**全てClient Componentsになる**ということです。Client Componentsはサーバーモジュールを`import`できない以上、これも当然の帰結です。

`"use client";`はこのように依存関係において境界(Boundary)を定義するもので、この境界はよく**Client Boundary**と表現されます。

:::message
よくある誤解のようですが、以下は**誤り**になります。注意しましょう。

- `"use client";`宣言付モジュールのコンポーネントだけがClient Components
- 全てのClient Componentsに`"use client";`が必要

:::

以下の記事ではReact Server Componentsにおけるコンポーネント間の関係性を、コンポーネントの「親」と「所有者」の違いから説明しています。

https://reacttraining.com/blog/react-owner-components

「親」はコンポーネントツリーにおける上位階層のコンポーネントを指し、「所有者」はReactElementを定義・実行するコンポーネントを指します。Client Componentsが「所有者」の場合、所有するコンポーネントは当然クライアントサイドで実行されるため全てClient Componentsである必要があります。一方単に「親」の場合、コンポーネントを実行する権限を持たないのでClient ComponentsとServer Componentsは親子関係を持つことができます。

## 設計・プラクティス

前述の通り、App RouterでServer Componentsの設計を活かすにはClient Componentsを独立した形に切り分けることが重要となります。

これには大きく以下2つの方法があります。

### コンポーネントツリーの末端をClient Componentsにする

1つは**コンポーネントツリーの末端をClient Componentsにする**というシンプルな方法です。Client Boundaryを下層に限定するとも言い換えられます。

例えば検索バーを持つヘッダーを実装する際に、ヘッダーごとClient Componentsにするのではなく検索バーの部分だけClient Componentsとして切り出し、ヘッダー自体はServer Componentsに保つといった方法です。

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

このようにClient Componentsをできるだけ小さく保てる境界で切り出すことは時に重要です。

### Compositionパターンを活用する

上記の方法はシンプルな解決策ですが、どうしても上位層のコンポーネントをClient Componentsにする必要がある場合もあります。その際には**Compositionパターン**を活用して、Client Componentsを分離することが有効です。

前述の通り、Client ComponentsはServer Componentsを`import`することができません。しかしこれは依存関係上の制約であって、コンポーネントツリーとしてはClient Componentsの`children`などのpropsにServer Componentsを渡すことで、レンダリングが可能です。

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

### 「後からComposition」の手戻り

Compositionパターンを駆使すればServer Components中心に部分的にClient Componentsを組み込むことが可能です。しかし、Client Componentsである程度画面の実装を進めて後からCompositionパターンを導入しようとすると、Client Componentsの設計を大幅に変更せざるを得なくなったり、Server Components中心な設計から逸脱してしまう可能性があります。

そのため、React Server Componentsにおいては設計する順番も非常に重要です。画面を実装する段階ではまずデータフェッチを行うServer Componentsを中心に設計し、そこに必要に応じてClient Componentsを末端に配置したりCompositionパターンで組み込んで実装を進めていくことを筆者はお勧めします。

この「データフェッチを行うServer Componentsを中心に設計」する際には、次章の[Container/Presentationalパターン](part_2_presentational_container_pattern)におけるContainer Componentsを組み立てることに等しい工程です。
