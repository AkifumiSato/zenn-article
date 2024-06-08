---
title: "PPRはアイランドアーキテクチャなのか"
emoji: "🏝️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react", "astro"]
published: true
---

先日、Next.jsの新たなレンダリングモデルである**Partial Pre-Rendering**(以降PPR)について記事を投稿しました。

https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering

この記事を書いてる時は意識してなかったのですが、感想でアイランドアーキテクチャに言及されるケースが散見されました。社内で上記記事の話題になった時も同様に、アイランドアーキテクチャとの違いについて問われました。

結論から言うと、**PPRとアイランドアーキテクチャは全く異なる**ものです。本稿ではPPRとアイランドアーキテクチャの違いについて解説します。

## PPR

まずはPPRとアイランドアーキテクチャの概要を改めて整理しましょう。

PPRは**ページをstatic renderingとしつつ、部分的にdynamic rendering**にすることが可能なレンダリングモデルです。具体的には、画面をbuild時(もしくはrevalidate後)に静的生成しつつ、リクエスト毎に処理が必要な一部動的な部分を遅延レンダリングにすることが可能になります。Next.js App RouterではレスポンスがStreamベースなので、PPRの場合まず静的な部分が早期に送信され、遅延レンダリングが完了するたびに徐々に動的な部分が送信されて**1HTTPレスポンス**で完結します。

以下は[Next.jsのドキュメント](https://rc.nextjs.org/learn/dashboard-app/partial-prerendering#what-is-partial-prerendering)で紹介されてるECサイトにおける商品ページの構成例です。

![ppr shell](/images/nextjs-partial-pre-rendering/ppr-shell.png)

具体的な挙動イメージは公式のDEMOページがわかりやすいので、下記にアクセスしてレコメンド情報やカート情報がスケルトンから置き換わる様子をぜひご覧ください。

https://www.partialprerendering.com/

PPRはNext.jsによって提唱され現在も開発中で、Next.js v15(RC)でexperimentalフラグ付きで利用可能です。Next.jsのPPRでは、`<Suspense>`境界をもってstatic/dynamic renderingを分けられます。

```tsx
import { Suspense } from "react";
import { PostFeed, Weather } from "./Components";

export default function Posts() {
  // Suspenseの外側はstatic rendering
  return (
    <section>
      <Suspense fallback={<p>Loading feed...</p>}>
        {/* PostFeedはdynamic rendering */}
        <PostFeed />
      </Suspense>
      <Suspense fallback={<p>Loading weather...</p>}>
        {/* Weatherはdynamic rendering */}
        <Weather />
      </Suspense>
    </section>
  );
}
```

:::message
より詳細な内容については[前述の記事](https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering)をご参照ください。
:::

## アイランドアーキテクチャ

一方**アイランドアーキテクチャ**(Islands Architecture)は[Astro](https://astro.build/)や[Fresh](https://fresh.deno.dev/)などで採用されてるクライアントサイドアーキテクチャです。これは**パーシャルハイドレーション**と呼ばれる、必要な部分のみをハイドレーションする手法をベースに構築されています。日本語ドキュメントだと、以下Astroのドキュメントがわかりやすいかと思います。

https://docs.astro.build/ja/concepts/islands/

アイランドアーキテクチャは画面全体を静的HTMLという広大な海に見立てて、独立したインタラクティブなUI郡を1つの島(**アイランド**)と捉えることで「広大な海にいくつかのアイランドが、独立して存在している様子」を比喩して命名されました。以下は[Preactの作者のブログ](https://jasonformat.com/islands-architecture/)より引用したアイランドアーキテクチャの例です。

![island architecture](/images/ppr-vs-islands-architecture/islands-architecture-example.png)

色がついてるところがアイランドです。`Header`にはハンバーガーメニューなど、`Sidebar`にはアコーディオンなどがあってインタラクティブということなのでしょう。カルーセルはインタラクティブな典型的要素です。色がついてないところはJavaScriptを必要としない静的なHTMLです。

## PPRとアイランドアーキテクチャの違い

PPRとアイランドアーキテクチャはどちらも「静的」な部分をベースに一部を分離するという点で似ているようにも見えます。前述の図も見比べると画面全体を静的とし、一部を分離している様子なども似ているかもしれません。

しかし、**PPRの主眼はサーバーサイドでのレンダリング、アイランドアーキテクチャの主眼はクライアントサイドでのレンダリングです**。同じく「静的」という言葉を用いていますが、これらは「何が静的か」という点で全く異なる意味を持っています。

| 観点        | PPR              | アイランドアーキテクチャ          |
|-----------|------------------|-----------------------|
| 主な視点      | サーバー             | クライアント                |
| 主な最適化対象   | TTFB             | JavaScriptサイズ         |
| 「静的」が指すもの | static rendering | JavaScriptを必要としないHMTL |

PPRで静的と表現されるのはstatic renderingです。一方アイランドアーキテクチャで静的と表現されるのはJavaScriptを必要としない非インタラクティブなHTMLです。

### Client Componentsとislandの比較

ここまでの説明を経てすでにお気づきの方もいるかもしれませんが、アイランドアーキテクチャにおけるアイランド相当な概念が、ReactやNext.jsの世界でも存在します。**Client Components**です。Server ComponentsにClient Componentsを埋め込んでいく様子は、まさしく静的HTMLの海に、アイランドを埋め込んでいく様子に近しいものです。

このような比較はDan Abramov氏も言及しています。

https://x.com/dan_abramov2/status/1757986886390264291

> - Server and Client components
> - Astro Templates and Astro Islands
> - PHP partials and jQuery plugins
> 
> these are all examples of two-layer architectures.

Server ComponentsとClient Components、Astro TemplatesとAstro Islands、PHP とjQuery pluginsなど、これらは全て2層のアーキテクチャです。外側のレイヤーはサーバー側で実行され、内側のレイヤーはクライアントサイドで(も)実行されます。

アイディア自体は新しいものではなく、多くで使われている古きものです。しかしこれらは同一ではありません。それぞれに小さな進化が含まれ、時代と共に技術の螺旋を歩んでいます。

https://speakerdeck.com/twada/understanding-the-spiral-of-technologies-2023-edition?slide=10

> - 技術の変化の歴史は一見すると**振り子**に見える
> - でも実は**螺旋**構造。同じところには戻ってこない
> - **差分**と、それを**可能にした技術**が重要

古くはPHPとjQuery pluginsで実装していたアイディアが今ではNext.jsやAstroで採用されている訳ですが、これらはそれぞれ異なる螺旋的進化を経ています。似ているからこそ違いを理解することが重要です。

## 感想

PPRではサーバー側のstatic/dynamicな境界を設計する必要があります。一方Server/Client Componentsではクライアント側でのインタラクティブ/非インタラクティブな境界を設計する必要があります。これら2層のレイヤーごとに設計を行う必要があることを意識すると、アイランドアーキテクチャとPPRは全く異なり、むしろServer/Client Componentsと類似した技術であることが理解いただけるかと思います。そしてこの2層のレイヤーに分けて考える必要があることは、前回の記事でも言及しましたuhyoさんの言う**多段階計算**そのものです。

https://zenn.dev/uhyo/articles/react-server-components-multi-stage#%E4%B8%80%E8%A8%80%E3%81%A7react-server-components%E3%82%92%E7%90%86%E8%A7%A3%E3%81%99%E3%82%8B

こうやって日々深堀っていくほどこの多段階計算の理解が重要だと感じます。そしてこの多段階計算の1段目(サーバー側)と2段目(クライアント側)を分けて考えられると、複雑そうに見えるNext.jsのアーキテクチャがシンプルに見えてくるかもしれません。

筆者なりにまとめてみましたが、わかりづらい点などあればご指摘いただけたら幸いです。
