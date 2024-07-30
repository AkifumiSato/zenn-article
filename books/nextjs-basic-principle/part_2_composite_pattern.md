---
title: "Compositionパターン"
---

## 要約

Compositionパターンを駆使して、Client Componentsを適切に切り分けましょう。

## 背景

[第1部](part_1)で述べたように、React Server Componentsのメリットを活かすにはServer Components中心の設計が重要となります。そのため**Client Componentsは適切に分離・独立した実装**となっていることが好ましいですが、Client Componentsには依存関係における重要な制約が存在します。

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

### Client Boundary

もう1つ注意すべきなのは、`"use client";`が記述されたモジュールから`import`されるモジュール以降は全て暗黙的にクライアントモジュールとして扱われ、それらで定義されたコンポーネントは**全てClient Componentsになる**ということです。Client Componentsはサーバーモジュールを`import`できない以上、これも当然の帰結です。

`"use client";`はこのように依存関係において境界(Boundary)を定義するもので、この境界はよく**Client Boundary**と表現されます。

:::message
よくある誤解ですが、以下は**誤り**なので注意しましょう。

- `"use client";`宣言付モジュールのコンポーネント**だけ**がClient Componentsになる
- 全てのClient Componentsに`"use client";`が必要

:::

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

前述の通り、App RouterでServer Componentsの設計を活かすにはClient Componentsを適切に切り分けることが重要となります。

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

このようにClient Componentsをできるだけ小さく保てる境界でコンポーネントを切り出せることが重要になってきます。

### Compositionパターンを活用する

上記の方法はシンプルな解決策ですが、どうしても上位のコンポーネントをClient Componentsにする必要がある場合もあります。その際には**Compositionパターン**を活用して、Client Componentsを分離することが有効です。

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

TBW
