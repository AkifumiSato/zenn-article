---
title: "SuspenseとStreaming"
---

## 要約

Dynamic Renderingで特に重いコンポーネントのレンダリングは`<Suspense>`で遅延させて、Streaming SSRにしましょう。

## 背景

[Dynamic Rendering↗︎](https://nextjs.org/docs/app/guides/caching#static-and-dynamic-rendering)ではRoute全体をレンダリングするため、[Dynamic RenderingとData Cache](part_3_dynamic_rendering_data_cache)ではData Cacheを活用することを検討すべきであるということを述べました。しかし、キャッシュできないようなデータフェッチに限って無視できないほど遅いということはよくあります。

## 設計・プラクティス

Next.jsでは[Streaming SSR↗︎](https://nextjs.org/docs/app/getting-started/linking-and-navigating#streaming)をサポートしているので、重いデータフェッチを伴うServer Componentsのレンダリングを遅延させ、ユーザーにいち早くレスポンスを返し始めることができます。具体的には、Next.jsは`<Suspense>`のfallbackを元に即座にレスポンスを送信し始め、その後、`<Suspense>`内のレンダリングが完了次第結果がクライアントへと続いて送信されます。

[並行データフェッチ](part_1_concurrent_fetch)で述べたようなデータフェッチ単位で分割されたコンポーネント設計ならば、`<Suspense>`境界を追加するのみで容易に実装できるはずです。

### 実装例

少々極端な例ですが、以下のような3秒の重い処理を伴う`<LazyComponent>`は`<Suspense>`によってレンダリングが遅延されるので、ユーザーは3秒を待たずにすぐにページのタイトルや`<Clock>`を見ることができます。

```tsx
import { setTimeout } from "node:timers/promises";
import { Suspense } from "react";
import { Clock } from "./clock";

export const dynamic = "force-dynamic";

export default function Page() {
  return (
    <div>
      <h1>Streaming SSR</h1>
      <Clock />
      <Suspense fallback={<>loading...</>}>
        <LazyComponent />
      </Suspense>
    </div>
  );
}

async function LazyComponent() {
  await setTimeout(3000);

  return <p>Lazy Component</p>;
}
```

_即座にレスポンスが表示される_
![Streaming SSR loading](/images/nextjs-basic-principle/streaming-ssr-loading.png)

_3秒後、遅延されたレンダリングが表示される_
![Streaming SSR rendered](/images/nextjs-basic-principle/streaming-ssr-rendered.png)

## トレードオフ

### fallbackのLayout Shift

Streaming SSRを活用するとユーザーに即座に画面を表示し始めることができますが、画面の一部にfallbackを表示しそれが後に置き換えられるため、いわゆる**Layout Shift**が発生する可能性があります。

置き換え後の高さが固定ならば、fallbackも同様の高さで固定することでLayout Shiftを防ぐことができます。一方、置き換え後の高さが固定でない場合にはLayout Shiftが発生することになり、[Time to First Byte↗︎](https://web.dev/articles/ttfb?hl=ja)(TTFB)と[CumulativeLayout Shift↗︎](https://web.dev/articles/cls?hl=ja)(CLS)のトレードオフが発生します。

そのため実際のユースケースにおいては、コンポーネントがどの程度遅いのかによってレンダリングを遅延させるべきかどうか判断が変わってきます。筆者の感覚論ですが、たとえば200ms程度のデータフェッチを伴うServer ComponentsならTTFBを短縮するよりLayout Shiftのデメリットの方が大きいと判断することが多いでしょう。1sを超えてくるようなServer Componentsなら迷わず遅延することを選びます。

TTFBとCLSどちらを優先すべきかはケースバイケースなので、状況に応じて最適な設計を検討しましょう。

### StreamingとSEO

SEOにおける古い見解では、以下のようなものがありました。

- 「Googleに評価されたいコンテンツはHTMLに含むべき」
- 「見えてないコンテンツは評価されない」

これらが事実なら、Streaming SSRはSEO的に不利ということになります。

しかし、Vercelが独自に行った大規模な調査によると、Streaming SSRした内容が**Googleに評価されないということはなかった**とのことです。

https://vercel.com/blog/how-google-handles-javascript-throughout-the-indexing-process

この調査によると、JS依存なコンテンツがindexingされる時間については以下のようになっています。

- 50%~: 10s
- 75%~: 26s
- 90%~: ~3h
- 95%~: ~6h
- 99%~: ~18h

Googleのindexingにある程度遅延がみられるものの、上記のindexing速度は十分に早いと言えるため、コンテンツの表示がJS依存なStreaming SSRがSEOに与える影響は少ない(もしくはほとんどない)と筆者は考えます。
