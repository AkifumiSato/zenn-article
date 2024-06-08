---
title: "PPRはアイランドアーキテクチャなのか"
emoji: "🏝️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

先日、Next.jsの新たなレンダリングモデルである**Partial Pre-Rendering**(以降PPR)について記事を投稿しました。

https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering

この記事を書いてる時は意識してなかったのですが、感想を見ているとアイランドアーキテクチャに言及されるケースが散見されました。社内でも同様に、アイランドアーキテクチャとの違いについて問われました。

結論から言うと、**PPRとアイランドアーキテクチャは全く異なる**ものです。本稿ではPPRとアイランドアーキテクチャの類似点と違いについて解説します。

## PPR

先日の記事や他の記事で解説は多数あると思いますが、まずはPPRとアイランドアーキテクチャの概要を改めて整理しましょう。

PPRは**ページをstatic renderingとしつつ、部分的にdynamic rendering**にすることが可能なレンダリングモデルです。具体的には、画面をbuild時(もしくはrevalidate後)に静的生成しつつ、画面の一部動的な部分を遅延レンダリングすることが可能になります。PPRのレスポンスStreamではまず静的な部分を送信し、遅延レンダリングが完了するたびに徐々に動的部分を送信します。以下はNext.jsのドキュメントで紹介されてるECサイトにおける商品ページの構成例です。

![ppr shell](/images/nextjs-partial-pre-rendering/ppr-shell.png)

具体的な挙動イメージは公式DEMOがわかりやすいので、下記にアクセスしてレコメンド情報やカートなどの情報がスケルトンから置き換わる様子をぜひご覧ください。

https://www.partialprerendering.com/

PPRはNext.jsによって提唱され現在も開発中で、Next.js v15(RC)で現在利用可能です。Next.jsのPPRでは、`<Suspense>`境界をもって静的・動的を分けられます。

```tsx
import { Suspense } from "react";
import { PostFeed, Weather } from "./Components";

export default function Posts() {
  return (
    <section>
      <Suspense fallback={<p>Loading feed...</p>}>
        <PostFeed />
      </Suspense>
      <Suspense fallback={<p>Loading weather...</p>}>
        <Weather />
      </Suspense>
    </section>
  );
}
```

より詳細な内容については[前述の記事](https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering)をご参照ください。

## アイランドアーキテクチャ

一方**アイランドアーキテクチャ**(Islands Architecture)は[Astro](https://astro.build/)や[Fresh](https://fresh.deno.dev/)などで採用されてる**クライアントサイド**アーキテクチャです。これは**パーシャルハイドレーション**と呼ばれる、必要な部分のみをハイドレーションする手法をベースに構築されています。日本語ドキュメントだと、以下Astroのドキュメントがわかりやすいかと思います。

https://docs.astro.build/ja/concepts/islands/

アイランドアーキテクチャの由来は**アイランド**(Islands)と呼ばれる独立したインタラクティブなUI郡を1つの島と捉え、画面全体は広大な静的HTMLの海に見立てることで「広大な海にいくつかの島が、独立して存在している様子」を比喩したものです。以下は[Preactの作者のブログ](https://jasonformat.com/islands-architecture/)より引用したアイランドアーキテクチャの例です。

![island architecture](/images/ppr-vs-islands-architecture/islands-architecture-example.png)

色がついてるところがアイランドです。`Header`にはハンバーガーメニューなど、`Sidebar`にはアコーディオンなどがあってインタラクティブなことが多いのでそれらがアイランドとして分離している様子がわかります。カルーセルはインタラクティブな典型的要素です。

## PPRとアイランドアーキテクチャの違い

PPRとアイランドアーキテクチャはどちらも「静的」な部分をベースに一部を分離するという点で似ているようにも見えます。前述の図も見比べると画面の一部を分離している様子なども似ているかもしれません。

しかし、**PPRの主眼はサーバー側でのレンダリング、アイランドアーキテクチャの主眼はクライアントサイドでのレンダリングです**。同じく「静的」という言葉を用いていますが、これらは「何が静的か」という点で全く異なる意味を持っています。

| 観点        | PPR              | アイランドアーキテクチャ            |
|-----------|------------------|-------------------------|
| 主な視点      | サーバー             | クライアント                  |
| 主な最適化対象   | TTFB             | JavaScriptサイズ           |
| 「静的」が指すもの | 静的(static)レンダリング | JavaScriptを必要としないHMTL要素 |

PPRで静的と表現されるのは静的(static)レンダリングです。一方アイランドアーキテクチャで静的と表現されるのはJavaScriptを必要としないHTML要素です。

### Client Componentsとislandの比較

ここまでの説明を経てすでにお気づきの方もいるかもしれませんが、アイランドアーキテクチャにおけるアイランド相当な概念が、ReactやNext.jsの世界でも存在します。**Client Components**です。Server ComponentsにClient Componentsを埋め込んでいく様子は、まさしくアイランドを埋め込んでいく様子に近しいものです。

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

## 感想

PPRではサーバー側のstatic/dynamicな境界を設計する必要があります。一方Server/Client Componentsではクライアント側でのインタラクティブ/非インタラクティブな境界を設計する必要があります。これら2層のレイヤーごとに設計を行う必要があることを意識すると、アイランドPPRとアーキテクチャは全く異なり、むしろServer/Client Componentsと類似した技術であることが理解いただけるかと思います。そしてこの2層のレイヤーに分けて考える必要があることは、前回の記事でも言及しましたuhyoさんの言う**多段階計算**そのものです。

https://zenn.dev/uhyo/articles/react-server-components-multi-stage#%E4%B8%80%E8%A8%80%E3%81%A7react-server-components%E3%82%92%E7%90%86%E8%A7%A3%E3%81%99%E3%82%8B

面白い記事だと思ってましたが、こうやって日々ふかぼっていくほどこの多段階計算の理解が重要だと日々感じています。そしてこの多段階計算の1段目(サーバー側)と2段目(クライアント側)を分けて考えられると、複雑そうに見えるNext.jsのアーキテクチャがシンプルに見えてくるかもしれません。

筆者なりにまとめてみましたが、わかりづらい点などあれば感想いただけたら幸いです。
