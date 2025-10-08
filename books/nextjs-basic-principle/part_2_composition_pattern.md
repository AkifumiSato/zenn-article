---
title: "Compositionパターン"
---

## 要約

Compositionパターンを駆使して、Server Componentsを中心に組み立てたコンポーネントツリーからClient Componentsを適切に切り分けましょう。

## 背景

[第1部 データフェッチ](part_1)で述べたように、RSCのメリットを活かすにはServer Components中心の設計が重要となります。そのため、Client Componentsは**適切に分離・独立**していることが好ましいですが、これを実現するにはClient Componentsの依存関係における以下2つの制約を考慮しつつ設計する必要があります。

:::message
以下は[クライアントとサーバーのバンドル境界](part_2_bundle_boundary)で解説した内容と重複します。
:::

### Client Bundleはサーバーモジュールを`import`できない

Client Bundle^[RSCにおいて、Client Componentsが含まれるバンドルを指します。]はServer Componentsはじめサーバーモジュールを`import`できません。

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

この制約に対し唯一例外となるのが`"use server"`が付与されたファイルや関数、つまり [Server Functions↗︎](https://ja.react.dev/reference/rsc/server-functions)です。

::::details Server Functionsの実装例

```ts :create-todo.ts
"use server";

export async function createTodo() {
  // サーバーサイド処理
}
```

```tsx :create-button.tsx
"use client";

import { createTodo } from "./create-todo"; // 💡Server Functionsならimportできる

export function CreateButton({ children }: { children: React.ReactNode }) {
  return <button onClick={createTodo}>{children}</button>;
}
```

:::message
Server FunctionsはClient Bundleから普通の関数のように実行することが可能ですが、実際には当然通信処理が伴うため、引数や戻り値には[Reactがserialize可能なもの↗︎](https://ja.react.dev/reference/rsc/use-server#serializable-parameters-and-return-values)のみを利用できます。
:::

::::

### Client Boundary

`"use client"`が記述されたClient Boundary^[サーバー -> クライアントのバンドル境界を指します。]となるモジュールから`import`されるモジュールとその子孫は、**暗黙的に全てClient Bundle**に含まれます。そのため、定義されたコンポーネントは全てClient Componentsとして実行可能でなければなりません。

:::message alert
以下はよくある誤解です。

##### Q. `"use client"`を宣言したモジュールのコンポーネントだけがClient Components？

`"use client"`はClient Boundaryを宣言するためのものであり、Client Bundleに含まれるコンポーネントは全てClient Componentsとして扱われます。

##### Q. 全てのClient Componentsに`"use client"`が必要？

Client Boundaryとして扱うことがないなら`"use client"`は不要です。Client Bundleに含まれることを保証したいなら、[`client-only`↗︎](https://www.npmjs.com/package/client-only)を利用しましょう。

:::

## 設計・プラクティス

前述の通り、RSCでServer Componentsの設計を活かすにはClient Componentsを独立した形に切り分けることが重要となります。

これには大きく以下2つの方法があります。

### コンポーネントツリーの末端をClient Componentsにする

1つは、コンポーネントツリーの**末端をClient Componentsにする**というシンプルな方法です。Client Boundaryを下層に限定するとも言い換えられます。

例えば検索バーを持つヘッダーの場合、ヘッダー全体ではなく検索バー部分をClient Boundaryとし、ヘッダー自体はServer Componentsに保つといった方法です。

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

上記の方法はシンプルな解決策ですが、どうしても上位のコンポーネントをClient Componentsにする必要がある場合もあります。その際には**Compositionパターン**を活用して、Client Componentsを分離することが有効です。

前述の通りClient BundleはServer Componentsを`import`することができませんが、これはモジュールツリーにおける制約であり、コンポーネントツリーとしてはClient Componentsの`children`などのpropsにServer Componentsを渡すことで、レンダリングが可能です^[参考: [公式ドキュメント↗︎](https://ja.react.dev/reference/rsc/use-client#why-is-copyright-a-server-component)]。

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

Compositionパターンを駆使すればServer Componentsを中心にしつつ、部分的にClient Componentsを組み込むことが可能です。しかし、上位のコンポーネントにClient Boundaryを宣言し、後からCompositionパターンを導入しようとすると、Client Componentsの設計を大幅に変更せざるを得なくなったりServer Components中心な設計から逸脱してしまう可能性があります。

このような手戻りを防ぐためのテクニックとして、次章では[UIをツリーに分解する](part_2_container_1st_design)設計手順について解説します。
