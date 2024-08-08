---
title: "Partial Pre Rendering(PPR)"
---

## 要約

PPRは従来のレンダリングモデルのメリットを組み合わせて、シンプルに整理した新しいアプローチです。外側をstatic rendering、`<Suspense>`境界内をdynamic renderingとすることが可能で、既存のモデルを簡素化しつつも高いパフォーマンスを実現します。PPRの使い方・考え方・実装状況を理解しておきましょう。

:::message alert
本稿はNext.js v15.0.0-rc.0時点の情報を元に執筆しており、PPRはさらにexperimentalな機能です。v15.0.0のリリース時や、PPRがstableな機能として提供される際には機能の一部が変更されてる可能性がありますので、ご注意下さい。
:::

:::message
本章は筆者の過去の記事の内容とまとめになります。より詳細にPPRについて知りたい方は以下をご参照ください。
https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering
:::

## 背景

従来Next.jsは[SSR](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering)・[SSG](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation)・[ISR](https://nextjs.org/docs/pages/building-your-application/data-fetching/incremental-static-regeneration)をサポートしてきました。App Routerではこれらに加え、[Streaming SSR](https://nextjs.org/docs/app/building-your-application/rendering/server-components#streaming)もサポートしています。複数のレンダリングモデルをサポートしているため付随するオプションが多数あり、複雑化している・考えることが多すぎるといったフィードバックがNext.js開発チームに多数寄せられていました。

App Routerはこれらをできるだけシンプルに整理するために、サーバー側でのレンダリングをstatic renderingとdynamic renderingという2つのモデルに再整理しました。

| レンダリング                                                                                                                   | タイミング            | Pages Routerとの比較 |
| ------------------------------------------------------------------------------------------------------------------------------ | --------------------- | -------------------- |
| [static rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default) | build時やrevalidate後 | SSG・ISR相当         |
| [dynamic rendering](https://nextjs.org/docs/app/building-your-application/rendering/server-components#dynamic-rendering)       | ユーザーリクエスト時  | SSR相当              |

しかし、v14までこれらのレンダリングの選択はページやレイアウト単位でしかできませんでした。そのため、大部分が静的化できるようなページでも一部動的なコンテンツがある場合には、ページ全体をdynamic renderingにするか、static rendering+クライアントサイドデータフェッチで処理する必要がありました。

## 設計・プラクティス

[Partial Pre Rendering(PPR)](https://nextjs.org/docs/app/api-reference/next-config-js/partial-prerendering)はこれらをさらに整理し、ページはstatic rendering、`<Suspense>`境界内をdynamic renderingとすることを可能としました。これにより、必ずしもレンダリングをページやレイアウト単位で考える必要はなくなり、1つのページ・1つのHTTPレスポンスにstaticとdynamicを混在させることができるようになりました。

以下は[公式チュートリアル](https://nextjs.org/learn/dashboard-app/partial-prerendering)からの引用画像です。

![ppr shell](/images/nextjs-partial-pre-rendering/ppr-shell.png)

レイアウトや商品情報についてはstatic renderingで構成されていますが、カートやレコメンドといったユーザーごとに異なるであろう部分はdynamic renderingとすることが表現されています。

### ユーザーから見たPPR

PPRでは、static renderingで生成されるhtmlやRSC Payloadに`<Suspense>`の`fallback`が埋め込まれます。`fallback`はdynamic renderingが完了するたびに置き換わっていくことになります。

そのため、ユーザーから見るとNext.jsサーバーは即座にページの一部分を返し始め、表示された`fallback`が徐々に置き換わっていくように見えます。

以下はレンダリングに3秒ほどかかるRandomなTodoを表示するページの例です。

_初期表示_
![stream start](/images/nextjs-partial-pre-rendering/ppr-stream-start.png)

_約 3 秒後_
![stream end](/images/nextjs-partial-pre-rendering/ppr-stream-end.png)

:::message
より詳細な挙動の説明は筆者の過去の記事で解説しているので、興味がある方は以下をご参照ください。
https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering#ppr%E3%81%AE%E6%8C%99%E5%8B%95%E8%A6%B3%E5%AF%9F
:::

### PPRの使い方

PPRは、本書執筆時点における最新のRCであるv15.0.0-rc.0でまだexperimentalな機能という位置付けです。そのため、PPRを利用するには`next.config.js`に以下の設定を追加する必要があります。

```ts
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

あとはdynamic renderingにしたい部分を`<Suspense>`でdynamic renderingの境界を定義することができます。

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

### PPRの今後

前述の通り、PPRはv14.x~v15.0.0(RC)においてまだexperimentalな機能です。PPRに伴うNext.js内部の変更は大規模なもので、バグや変更される挙動もあるかもしれません。実験的利用以上のことは避けておくのが無難でしょう。

ただし、PPRはNext.jsコアチームが本書執筆時現在、最も意欲的に取り組んでいる機能です。将来的には主要な機能となる可能性が高いので、先行して学んでおく価値はあると筆者は考えます。

### CDNキャッシュとの相性の悪さ

PPRではstaticとdynamicを混在させつつも、1つのHTTPレスポンスで完結するという特徴を持っています。これはレスポンス単位でキャッシュすることを想定したCDNとは非常に相性が悪いため、PPRはHTTPラウンドトリップを1回で済ませる一方でCDNキャッシュできないというトレードオフが発生します。
