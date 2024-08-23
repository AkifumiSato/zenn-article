---
title: "Compositionパターン"
---

## 要約

Compositionパターンを駆使して、Server Componentsを中心に組み立てたコンポーネントツリーからClient Componentsを適切に切り分けましょう。

## 背景

[_第1部 データフェッチ_](part_1)で述べたように、React Server Componentsのメリットを活かすにはServer Components中心の設計が重要となります。そのため、Client Componentsは**適切に分離・独立**していることが好ましいですが、これを実現するにはClient Componentsの依存関係における2つの制約を考慮しつつ設計する必要があります。

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

この制約に対し唯一例外となるのが`"use server";`が付与されたファイルや関数、つまり [Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)です。

```ts :actions.ts
"use server";

export async function create() {
  // サーバーサイド処理
}
```

```tsx :create-button.tsx
"use client";

import { create } from "./actions"; // 💡Server Actionsならimportできる

export function CreateButton({ children }: { children: React.ReactNode }) {
  return <button onClick={create}>{children}</button>;
}
```

### Client Boundary

もう1つ注意すべきなのは、`"use client";`が記述されたモジュールから`import`されるモジュール以降は、**全て暗黙的にクライアントモジュールとして扱われる**ということです。そのため、定義されたコンポーネントは全てClient Componentsとして実行可能でなければなりません。Client Componentsはサーバーモジュールを`import`できない以上、これも当然の帰結です。

`"use client";`はこのように依存関係において境界(Boundary)を定義するもので、この境界はよく**Client Boundary**と表現されます。

:::message
よくある誤解のようですが、以下は**誤り**なので注意しましょう。

- `"use client";`宣言付モジュールのコンポーネントだけがClient Components
- 全てのClient Componentsに`"use client";`が必要

:::

## 設計・プラクティス

前述の通り、App RouterでServer Componentsの設計を活かすにはClient Componentsを独立した形に切り分けることが重要となります。

これには大きく以下2つの方法があります。

### コンポーネントツリーの末端をClient Componentsにする

1つは**コンポーネントツリーの末端をClient Componentsにする**というシンプルな方法です。Client Boundaryを下層に限定するとも言い換えられます。

例えば検索バーを持つヘッダーを実装する際に、ヘッダーごとClient Componentsにするのではなく検索バーの部分だけClient Componentsとして切り出し、ヘッダー自体はServer Componentsに保つといった方法です。

```tsx :header.tsx
import { SearchBar } from "./search-bar"; // Client Components

// page.tsxなどのServer Componentsから利用される
export function Header() {
  return (
    <header>
      <h1>My App</h1>
      <SearchBar />
    </header>
  );
}
```

### Compositionパターンを活用する

上記の方法はシンプルな解決策ですが、どうしても上位層のコンポーネントをClient Componentsにする必要がある場合もあります。その際には**Compositionパターン**を活用して、Client Componentsを分離することが有効です。

前述の通り、Client ComponentsはServer Componentsを`import`することができません。しかしこれは依存関係上の制約であって、コンポーネントツリーとしてはClient Componentsの`children`などのpropsにServer Componentsを渡すことで、レンダリングが可能です。

前述の`<SideMenu>`の例を書き換えてみます。

```tsx :side-menu.tsx
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

```tsx :page.tsx
import { UserInfo } from "./user-info"; // Server Components
import { SideMenu } from "./side-menu"; // Client Components

/**
 * Client Components(`<SideMenu>`)の子要素として
 * Server Components(`<UserInfo>`)を渡せる
 */
export function Page() {
  return (
    <div>
      <SideMenu>
        <UserInfo />
      </SideMenu>
      <main>{/* ... */}</main>
    </div>
  );
}
```

`<SideMenu>`の`children`がServer Componentsである`<UserInfo />`となっています。これがいわゆるCompositionパターンと呼ばれる実装パターンです。

## トレードオフ

### 「後からComposition」の手戻り

Compositionパターンを駆使すればServer Componentsを中心にしつつ、部分的にClient Componentsを組み込むことが可能です。しかし、早期にClient Boundaryを形成し後からCompositionパターンを導入しようとすると、Client Componentsの設計を大幅に変更せざるを得なくなったり、Server Components中心な設計から逸脱してしまう可能性があります。

そのため、React Server Componentsにおいては設計する順番も非常に重要です。画面を実装する段階ではまずデータフェッチを行うServer Componentsを中心に設計し、そこに必要に応じてClient Componentsを末端に配置したりCompositionパターンで組み込んで実装を進めていくことを筆者はお勧めします。

データフェッチを行うServer Componentsを中心に設計することは、次章の[_Container/Presentationalパターン_](part_2_container_presentational_pattern)におけるContainer Componentsを組み立てることに等しい工程です。
