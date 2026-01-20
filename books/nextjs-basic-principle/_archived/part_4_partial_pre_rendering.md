---
title: "[Experimental] Partial Pre Rendering(PPR)"
---

## 要約

PPRは従来のレンダリングモデルのメリットを組み合わせて、シンプルに整理した新しいアプローチです。`<Suspense>`境界の外側をStatic Rendering、内側をDynamic Renderingとすることが可能で、既存のモデルを簡素化しつつも高いパフォーマンスを実現します。PPRの使い方・考え方・実装状況を理解しておきましょう。

:::message alert
本稿はNext.js v15.1.x時点の情報を元に執筆しており、PPRはさらにexperimentalな機能です。PPRがstableな機能として提供される際には機能の一部が変更されてる可能性がありますので、ご注意下さい。
:::

:::message
本章は筆者の過去の記事の内容とまとめになります。より詳細にPPRについて知りたい方は以下をご参照ください。
https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering
:::

## 背景

従来Next.jsは[SSR↗︎](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering)・[SSG↗︎](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation)・[ISR↗︎](https://nextjs.org/docs/pages/building-your-application/data-fetching/incremental-static-regeneration)をサポートしてきました。App Routerではこれらに加え、[Streaming SSR↗︎](https://nextjs.org/docs/app/getting-started/linking-and-navigating#streaming)もサポートしています。複数のレンダリングモデルをサポートしているため付随するオプションが多数あり、複雑化している・考えることが多すぎるといったフィードバックがNext.js開発チームに多数寄せられていました。

App Routerはこれらをできるだけシンプルに整理するために、サーバー側でのレンダリングをStatic RenderingとDynamic Renderingという2つのモデルに再整理しました^[参考: [公式ドキュメント↗︎](https://nextjs.org/docs/app/getting-started/linking-and-navigating#server-rendering)]。

| レンダリング          | タイミング            | Pages Routerとの比較 |
| --------------------- | --------------------- | -------------------- |
| **Static Rendering**  | build時やrevalidate後 | SSG・ISR相当         |
| **Dynamic Rendering** | ユーザーリクエスト時  | SSR相当              |

しかし、v14までこれらのレンダリングはRoute単位(`page.tsx`や`layout.tsx`)でしか選択できませんでした。そのため、大部分が静的化できるようなページでも一部動的なコンテンツがある場合には、ページ全体をDynamic Renderingにするか、Static Rendering+Client Componentsによるクライアントサイドデータフェッチで処理する必要がありました。

## 設計・プラクティス

[Partial Pre Rendering(PPR)↗︎](https://nextjs.org/docs/app/api-reference/next-config-js/partial-prerendering)はこれらをさらに整理し、基本はStatic Rendering、`<Suspense>`境界内をDynamic Renderingとすることを可能としました。これにより、必ずしもレンダリングをRoute単位で考える必要はなくなり、1つのページ・1つのHTTPレスポンスにStaticとDynamicを混在させることができるようになりました。

以下は[公式チュートリアル↗︎](https://nextjs.org/learn/dashboard-app/partial-prerendering)からの引用画像です。

![ppr shell](/images/nextjs-partial-pre-rendering/ppr-shell.png)

レイアウトや商品情報についてはStatic Renderingで構成されていますが、カートやレコメンドといったユーザーごとに異なるであろう部分はDynamic Renderingとすることが表現されています。

### ユーザーから見たPPR

PPRでは、Static Renderingで生成されるHTMLやRSC Payloadに`<Suspense>`の`fallback`が埋め込まれます。`fallback`はDynamic Renderingが完了するたびに置き換わっていくことになります。

そのため、ユーザーから見るとStreaming SSR同様、Next.jsサーバーは即座にページの一部分を返し始め、表示された`fallback`が徐々に置き換わっていくように見えます。

以下はレンダリングに3秒ほどかかるRandomなTodoを表示するページの例です。

_初期表示_
![stream start](/images/nextjs-partial-pre-rendering/ppr-stream-start.png)

_約 3 秒後_
![stream end](/images/nextjs-partial-pre-rendering/ppr-stream-end.png)

:::message
より詳細な挙動の説明は筆者の過去の記事で解説しているので、興味がある方は以下をご参照ください。
https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering#ppr%E3%81%AE%E6%8C%99%E5%8B%95%E8%A6%B3%E5%AF%9F
:::

### PPR実装

開発者がPPRを利用するには、Dynamic Renderingの境界を`<Suspense>`で囲むのみです。非常にシンプルかつReactのAPIを用いた実装であることも、PPRの優れている点です。

```tsx
import { Suspense } from "react";
import { StaticComponent, DynamicComponent, Fallback } from "@/app/ui";

export const experimental_ppr = true;

export default function Page() {
  return (
    <>
      <StaticComponent />
      <Suspense fallback={<Fallback />}>
        <DynamicComponent />
      </Suspense>
    </>
  );
}
```

## トレードオフ

### Experimental

PPRは、本書執筆時点でまだexperimentalな機能という位置付けです。そのため、PPRを利用するにはNext.jsの`canary`バージョンと、`next.config.ts`に以下の設定を追加する必要があります。

```ts :next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  experimental: {
    ppr: "incremental", // v14.xではboolean
  },
};

export default nextConfig;
```

上記を設定した上で、ページやレイアウトなどPPRを有効化したいモジュールで`experimental_ppr`をexportします。

```tsx
export const experimental_ppr = true;
```

### PPRの今後

前述の通り、PPRはまだexperimentalな機能です。PPRに伴うNext.js内部の変更は大規模なもので、バグや変更される挙動もあるかもしれません。現時点では、実験的利用以上のことは避けておくのが無難でしょう。

ただし、PPRはNext.jsコアチームが最も意欲的に取り組んでいる機能の1つです。将来的には主要な機能となる可能性が高いので、先行して学んでおく価値はあると筆者は考えます。

### CDNキャッシュとの相性の悪さ

PPRではStaticとDynamicを混在させつつも、1つのHTTPレスポンスで完結するという特徴を持っています。これはレスポンス単位でキャッシュすることを想定したCDNとは非常に相性が悪く、CDNキャッシュできないというトレードオフが発生します。

いずれは、Cloudflare WorkersなどのエッジコンピューティングからStatic Renderingな部分を返しつつ、オリジンからDynamic Renderingな部分を返すような構成が容易にできるような未来が来るかもしれません。今後のCDNベンダーやNext.jsチームの動向に期待したいところです。
