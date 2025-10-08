---
title: "Client Componentsのユースケース"
---

## 要約

Client Componentsを使うべき代表的なユースケースを覚えておきましょう。

- [#クライアントサイド処理](#クライアントサイド処理)
- [#サードパーティコンポーネント](#サードパーティコンポーネント)
- [#RSC Payload転送量の削減](#rsc-payload転送量の削減)

## 背景

[第1部 データフェッチ](part_1)では、データフェッチ観点を中心にServer Componentsの設計パターンについて解説してきました。Next.jsにおけるコンポーネント全体の設計は、Server Componentsを中心とした設計にClient Componentsを適切に組み合わせていく形で行います。

そのためには、そもそもいつClient Componentsにオプトインすべきなのか適切に判断できることが重要です。

## 設計・プラクティス

筆者がClient Componentsを利用すべきだと考える代表的な場合は大きく以下の3つです。

### クライアントサイド処理

最もわかりやすくClient Componentsが必要な場合は、クライアントサイド処理を必要とする場合です。以下のような場合が考えられます。

- `onClick()`や`onChange()`といったイベントハンドラの利用
- 状態hooks(`useState()`や`useReducer()`など)やライフサイクルhooks(`useEffect()`など)の利用
- ブラウザAPIの利用

```tsx
"use client";

import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  );
}
```

### サードパーティコンポーネント

Client Componentsを提供するサードパーティライブラリがRSCに未対応な場合は、利用者側でClient Boundaryを明示しなければならないことがあります。この場合は`"use client"`を指定してre-exportするか、利用者側で`"use client"`を指定する必要があります。

```tsx :app/_components/accordion.tsx
"use client";

import { Accordion } from "third-party-library";

export default Accordion;
```

```tsx :app/_components/side-bar.tsx
"use client";

import { Accordion } from "third-party-library";

export function SideBar() {
  return (
    <div>
      <Accordion>{/* ... */}</Accordion>
    </div>
  );
}
```

### RSC Payload転送量の削減

3つ目は[RSC Payload↗︎](https://nextjs.org/docs/app/getting-started/server-and-client-components#on-the-server)の転送量を減らしたい場合です。Client Componentsは当然ながらクライアントサイドでも実行されるので、Client Componentsが多いほどJavaScriptバンドルサイズは増加します。一方Server ComponentsはRSC Payloadとして転送されるため、Server ComponentsがレンダリングするReactElementや属性が多いほど転送量が多くなります。

つまり、Client ComponentsのJavaScript転送量とServer ComponentsのRSC Payload転送量は**トレードオフ**になります。

Client Componentsを含むJavaScriptバンドルは1回しかロードされませんが、Server ComponentsはレンダリングされるたびにRSC Payloadが転送されます。そのため、繰り返しレンダリングされるコンポーネントはRSC Payloadの転送量を削減する目的でClient Componentsにすることが望ましい場合があります。

例えば以下の`<Product>`について考えてみます。

```tsx
export async function Product() {
  const product = await fetchProduct();

  return (
    <div class="... /* 大量のtailwindクラス */">
      <div class="... /* 大量のtailwindクラス */">
        <div class="... /* 大量のtailwindクラス */">
          <div class="... /* 大量のtailwindクラス */">
            {/* `product`参照 */}
          </div>
        </div>
      </div>
    </div>
  );
}
```

hooksなども特になく、ただ`product`を取得・参照しているのみです。しかしこのデータを参照してるReactElementの出力結果サイズが大きいと、RSC Payloadの転送コストが大きくなりパフォーマンス劣化を引き起こす可能性があります。特に低速なネットワーク環境においてページ遷移を繰り返す際などには影響が顕著になりがちです。

このような場合においてはServer Componentsではデータフェッチのみを行い、ReactElement部分はClient Componentsに分離することでRSC Payloadの転送量を削減することができます。

```tsx
export async function Product() {
  const product = await fetchProduct();

  return <ProductPresentaional product={product} />;
}
```

```tsx
"use client";

export function ProductPresentaional({ product }: { product: Product }) {
  return (
    <div class="... /* 大量のtailwindクラス */">
      <div class="... /* 大量のtailwindクラス */">
        <div class="... /* 大量のtailwindクラス */">
          <div class="... /* 大量のtailwindクラス */">
            {/* `product`参照 */}
          </div>
        </div>
      </div>
    </div>
  );
}
```

## トレードオフ

### Client Boundaryと暗黙的なClient Components

[クライアントとサーバーのバンドル境界](part_2_bundle_boundary)で解説したように`"use client"`はバンドル境界を定義するものであり、Client Bundleに含まれるコンポーネントは**全てClient Components**として扱われます。

上位のコンポーネントでClient Boundaryを宣言してしまうと下層でServer Componentsを含むことができなくなってしまい、RSCのメリットをうまく享受できなくなってしまうケースが散見されます。このようなケースへの対応は次章の[Compositionパターン](part_2_composition_pattern)で解説します。

### Server ComponentsからClient Componentsへ渡せるProps

Server ComponentsはClient Componentsを含むことができますが、これはServer BundleからClient Bundleへとバンドル境界を跨ぐため、渡せるPropsは、[Reactがserialize可能なもの↗︎](https://ja.react.dev/reference/rsc/use-client#serializable-types)に限られます。

```tsx
export async function MyServerComponent() {
  // ...

  return <ClientComponent data={/* Reactがserialize可能なもののみ渡せる */} />;
}
```
