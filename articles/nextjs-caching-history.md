---
title: "Next.js Cache回想"
emoji: "🔄"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: true
---

:::message
この記事は[JSConf JP 2025で発表した内容](https://jsconf.jp/2025/en/talks/nextjs-caching-re-architecture)を、記事として執筆しなおしたものです。
:::

Next.jsは従来より、デフォルトで高いパフォーマンスを実現するフレームワークであることを重視してきました。App RouterにおいてもSSR・Streaming・Cacheなど様々な最適化により、高いパフォーマンスの実現を目指してきました。一方で、積極的なCache活用や複雑すぎるCacheの影響範囲は、開発者に多くの混乱をもたらしました。

v16で導入された**Cache Components**は、Next.jsのCacheのメンタルモデルを大きく変更するものであり、これまでの多くの課題を解決しうるリアーキテクチャです。この記事では、Cache Componentsが何を解決しようとしているのか、なぜリアーキテクチャに至ったのかなどの歴史的経緯を解説します。

## Next.jsの歴史

Next.jsには現在、**Pages Router**と**App Router**という2つのRouterが同梱されています^[参考: [App Router and Pages Router](https://nextjs.org/docs#app-router-and-pages-router)]。Pages Routerは2016年のNext.js公開当初から存在するRouterで、文字通りPageを中心としたRoutingを提供しています。SSRサポートに加え、SSGサポートやISRの発明など静的化によるパフォーマンス最適化を積極的に行ってきました。一方、App Routerは2023年にStableとなった新たなRouterで、[React Server Components](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md)のサポートやコロケーション志向な規約設計、積極的なCache活用などを特徴としています。

| 年      | 概要            | 詳細                                |
| ------- | --------------- | ----------------------------------- |
| 2013/05 | Reactが公開     |                                     |
| 2016/10 | Next.jsが公開   |                                     |
| 2018~   | Gatsby.jsの台頭 | SSGが注目される                     |
| 2019/07 | Next.js@v9.0    | dynamicルーティング・Typescript対応 |
| 2020/03 | Next.js@v9.3    | SSG対応                             |
| 2020/07 | Next.js@v9.5    | ISR対応                             |
| 2022/05 | Layout RFC発表  | 後のApp Router                      |
| 2022/10 | Next.js@v13.0   | App Router(Beta)                    |
| 2023/05 | Next.js@v13.4   | App Router(Stable)                  |
| 2024/10 | Next.js@v15.0   | `params`などの破壊的変更、PPRなど   |
| 2025/10 | Next.js@v16.0   | Cache Components                    |

Pages Routerは現在でも利用可能ですが、Next.jsにおける新規の機能開発などは行われておらず、現在Next.jsを使った開発はApp Routerが主流^[参考: [Vercelのテックリードのツイート](https://x.com/timneutkens/status/1982905913741644089)]となりつつあります。

## Legacy Cache: App Router初期のCache（v13~v14）

App Routerにおいても、Pages Routerで培われた静的化によるパフォーマンス最適化思想は引き継がれています。従来SSGやISRと呼ばれたページの静的化は、Cacheの1種として位置付けられ、より高度な最適化のために4層のCacheが導入されました。

![Next.jsのCacheの多層化](/images/nextjs-caching-history/cache-layer.png)

以下は[公式の表](https://nextjs.org/docs/app/guides/caching#overview)を翻訳したものです。

| Mechanism               | What              | Where  | Purpose                                    | Duration                                |
| ----------------------- | ----------------- | ------ | ------------------------------------------ | --------------------------------------- |
| **Request Memoization** | APIレスポンスなど | Server | React Component treeにおけるデータの再利用 | リクエストごと                          |
| **Data Cache**          | APIレスポンスなど | Server | ユーザーやデプロイをまたぐデータの再利用   | 永続的 (revalidate可)                   |
| **Full Route Cache**    | HTMLやRSC payload | Server | レンダリングコストの最適化                 | 永続的 (revalidate可)                   |
| **Router Cache**        | RSC Payload       | Client | ナビゲーションごとのリクエスト削減         | ユーザーセッション・時間 (revalidate可) |

これらのCacheを使いこなすことで高いパフォーマンスが期待できる一方、多層のCacheはそれ自体が高い認知負荷を伴うため、開発者に大きな負担を強いることとなりました。

当時筆者がCache周りで課題に感じていた点を、いくつか例示します。

### 課題1: 予想困難なデータフェッチ

Webアプリケーション開発において、Cacheできないデータフェッチを実装することは非常に一般的です。App Router初期においては、デフォルトで積極的なCache活用戦略が採用されていたため、データフェッチがCacheされないようにするにはOpt-outが必要でした。しかし、Opt-outの手段も複数あり、影響範囲が広範に及んだため、開発者は**多層のCacheと複数のOpt-out手段を考慮**する必要がありました。

下記の実装例では、リクエストごとにランダムなTodoを取得することを期待して`await getRandomTodo()`を実装しています。

```tsx
export default async function Page() {
  const { todo } = await getRandomTodo();
  // ...
}
```

しかし、このコードの読み手が`await getRandomTodo()`のCache挙動について正しく理解するには、多岐にわたるコードや設定を参照する必要があります。具体的には、`getRandomTodo()`の中の実装と外の実装どちらも確認しないと、Opt-outされるかどうか判断できませんでした。

- `getRandomTodo()`内で呼び出しているであろう`fetch()`で、`cache: "no-store"`や`next: { revalidate: 0 }`がオプション指定されているかどうか
- `getRandomTodo()`の内または外で、`cookies()`や`headers()`などを利用しているかどうか
- Pageや祖先のLayoutでCacheに関する[Route Segment Config](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config)が設定されているかどうか

このように、Full Route CacheやData Cacheといったサーバー側のCacheがOpt-outされるかどうかを実装から判断するには、広範なコード理解とNext.jsのCache仕様理解を必要としたため、開発者に大きな負担を強いました。

### 課題2: 複雑なCache設定

前述の通り、Cache挙動はRoute Segment Configで制御することが可能でしたが、これらの設定は高度な理解を必要とし、かつ影響範囲も広範なため、高い認知負荷を伴うこととなりました。

```ts
export const dynamic = "auto";
// 'auto' | 'force-dynamic' | 'error' | 'force-static'
export const fetchCache = "auto";
// 'auto' | 'default-cache' | 'only-cache'
// 'force-cache' | 'force-no-store' | 'default-no-store' | 'only-no-store'
export const revalidate = false;
// false | 0 | number
// ...
```

これらがLayoutに設定されていると下層のRoute全てに影響するため、開発者は影響範囲を慎重に設計する必要がありました。また、これらの設定によって末端のコンポーネントも挙動に影響を受けるため、開発者はコンポーネントとPageの依存関係を意識する必要があり、可読性や予測性を損なうこととなりました。

ダッシュボードのような非常に動的なアプリケーションの開発においてはCacheを一律Opt-outすることも多く、筆者の主観ではこれらの設定を「駆使する」より「避ける」方がベストプラクティスとなっていたと思います。

### 課題3: 不透明な仕様

App Router初期におけるCacheは「フレームワーク側が意識するものであって開発者が意識する必要はない」という価値観で設計されていました。そのため、前述の4層のCacheの存在や各Cacheの詳細な仕様は公式ドキュメントでは明示されておらず、開発者がNext.jsの仕様を把握するには詳細な検証やコードリーディングが必要でした。

ここでは当時筆者が特に問題視していた、Router Cacheの有効期限に関する仕様について紹介します。

#### Router Cacheの有効期限

当初のRouter Cacheは、StaticなRouteは5m、DynamicなRouteは30s保持され、これを更新するには**全Router Cacheを命令的に破棄**するしかありませんでした。また、これらの時間についてはハードコーディングされているため設定することもできず、issueでは[patch-package](https://www.npmjs.com/package/patch-package)でpatchする人も現れました。

フレームワーク初期におけるバグの多さも相まって、多くの開発者はこの挙動がバグなのか仕様なのか判別できず、大きな混乱を招きました。

::::details 余談: JSConf JP(2023)での発表

筆者はJSConf JP(2023)において、Next.jsのアンドキュメントな仕様や問題点について解説しました。

https://jsconf.jp/2023/talk/akfm-sato-1/
::::

## Cache Improvement: 議論と改善（v14~v15）

初期のApp RouterにおけるCacheは、動的なWebアプリケーションの開発者にとってともかく悩みの種でした。Next.js開発チームには、SNSやissueで多数のフィードバックが寄せられましたが、開発チームの想定とコミュニティの需要には一定認識の乖離が見られました。

このような状況を解消すべく、Next.js開発チームはDiscussionを作成し、そこで開発チームの描いてる世界観とコミュニティの需要をすり合わせ始めました。

https://github.com/vercel/next.js/Discussions/54075

このDiscussionは冒頭から非常に長い説明で始まっており、また、コミュニティからの質問や要望に対しても非常に丁寧に回答しており、丁寧なコミュニケーションを心がけている様子が伺えます。「コミュニティが求めてるユースケースは何か？」、「Router Cacheの寿命は設定できれば解決するのか？」「他の解決方法はないのか？」など、さまざまな議論が重ねられました。

このDiscussionはじめNext.js開発チームは改善方針を慎重に検討し、v14~15にかけて様々な改善を実施しました。以下は、v14~15で行われた改善の一部です。

### 改善1: `fetch()`のデフォルトCache廃止

v15では、`fetch()`によるData Cacheがデフォルトで無効化されました。これは破壊的変更でしたが、従来`fetch()`がデフォルトでCacheすることについて多くの混乱が見られたため、筆者はこれをとても大きな改善だったと捉えています。

一方で、Full Route CacheをOpt-outしてDynamic Renderingするには、引き続き`fetch()`に`cache: "no-store"`を指定する必要がありました。この点において、`fetch()`のデフォルトの振る舞いは引き続き開発者に混乱を招いたため、**改善はしたが解消はしなかった**と筆者は感じています。

```ts
const res = await fetch(`https://...`, {
  // default: Data Cacheは無効、ただしDynamic Renderingには**ならない**
  // "no-store": Data Cacheは無効、Dynamic Renderingを強制
  // "force-cache": Data Cacheは有効
  cache: "force-cache",
});
```

::::details デフォルトの変更に踏み切った背景: PPRの発見

前述の通り、Next.jsはデフォルトで高いパフォーマンスを実現するフレームワークであることを重視してきました。デフォルトでCache活用する戦略の変更は、この価値観に相反する可能性があり、当初Next.js開発チームは慎重な姿勢でした。しかし、Partial Pre-Rendering（PPR）の発見が状況を大きく変えました。

PPRの詳細については、筆者の過去の記事をご参照ください。

https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering

::::

### 改善2: ドキュメントの改善

前述の通り、App Router当初はCacheに関するドキュメントが不足していたため、仕様を理解するには挙動の観測とコードリーディングが必要でした。前述のDiscussionなどを経て、公式ドキュメントに[Caching in Next.js](https://nextjs.org/docs/app/guides/caching)が追加されました。これにより、Next.js開発チームが描いてるCache戦略の詳細が明示され、透明性が向上しました。

- 4層のCacheの明記
- 各Cacheの役割
- Static Rendering, Dynamic Renderingの定義と条件

### 改善3: staleTimesオプションとデフォルト値の変更

前述の通り、App Routerは当初Router Cacheの有効期限をハードコーディングしており、開発者は任意の値を設定できませんでしたが、これを設定可能にするオプションとしてv14.2で`staleTimes`が追加されました。

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  experimental: {
    staleTimes: {
      dynamic: 30,
      static: 180,
    },
  },
};

export default nextConfig;
```

また、DynamicなRouteが30s Cacheされることについても多くの混乱を招いたため、v15で`staleTimes.dynamic`のデフォルトは30sから0sに変更されました。

これによりRouter CacheはOpt-out型からOpt-in型へと変更され、デフォルトでRouter Cacheが積極的に活用されることにより招いていた混乱は解消されました。

## Cache Re-Architecture: 根本的な改善（v15~v16）

Next.jsはv14~v15で着実にCacheの改善を重ねました。しかし、多層のCacheや予想困難な影響範囲など、根本的な複雑さは解消されませんでした。

| v13の課題                       | v14~v15での改善                                          | 残った課題                                     |
| ------------------------------- | -------------------------------------------------------- | ---------------------------------------------- |
| 課題1: 予想困難なデータフェッチ | 改善1: `fetch()`のデフォルトCache廃止                    | 改善したが、依然として予想困難なデータフェッチ |
| 課題2: 複雑なCache設定          | -                                                        | 複雑なCache設定の理解と設計難易度              |
| 課題3: 不透明な仕様             | 改善2: ドキュメントの改善<br>改善3: staleTimesオプション | -                                              |

これらの解消には破壊的変更を伴うリアーキテクチャが必要だと思われました。そこで打ち出されたのが`"use cache"`でキャッシュを宣言する世界観です。

```tsx
// Function Level
export async function getData() {
  "use cache";

  return fetch("...").then((res) => res.json());
}

// Component Level
export default async function Page() {
  "use cache";

  return (
    <>
      <h1>Title</h1>
      <p>...</p>
    </>
  );
}
```

`"use cache"`は2024年のNext Confや公式ブログ[Our Journey with Caching](https://nextjs.org/blog/our-journey-with-caching)で発表されました。発表当時はPPR・`"use cache"`・Dynamic IO^[Dynamic IO: 動的処理を伴うコンポーネントには`<Suspense>`が必須となる世界観]がそれぞれ打ち出されており、最終的な世界観が不明瞭でしたが、2025年に発表されたv16ではこれらを統合する形で[Cache Components](https://nextjs.org/docs/app/getting-started/cache-components)フラグが導入されました。

### `"use cache"`とRSCの世界観

`"use client"`や`"use server"`はRSCの仕様として定義されているディレクティブですが、`"use cache"`はNext.js独自^[将来的には[vite-plugin-react-use-cache](https://github.com/jacob-ebey/vite-plugin-react-use-cache)など、Next.jsの`"use cache"`相当の機能が他フレームワークで利用可能になる可能性があります。]のディレクティブです。言い換えると、Next.jsは`"use cache"`によって、RSCの世界観を拡張していると言えます。

ReactチームのDan Abramov氏は[ブログ記事](https://overreacted.io/what-does-use-client-do/#two-worlds-two-doors)にて、RSCの世界観を「_2つの世界、2つのドア_」と説明しています。ServerとClient、2つの世界を行き来するためのドアが`"use client"`と`"use server"`です。

![RSC Door](/images/nextjs-caching-history/rsc-door.png =450x)

`"use cache"`も同様に、**Cacheの世界とドア**を定義してるものと看做すことができます。

![Cache Components](/images/nextjs-caching-history/cache-door.png)

このように、`"use cache"`にはRSCの設計思想と類似性を見出すことができます。また、`"use cache"`が宣言されたコンポーネントがComposable^[Reactチームは昨今、Composable=合成可能な設計思想を重視しています。]であることも、RSCの設計思想と一致していると言えます。

```tsx:page.tsx
export default async function PostPage(props: { params: Promise<{ id: string }> }) {
  const { id } = await props.params;

  return (
    {/* <PostContent>: Cached (`"use cache"`) */}
    <PostContent id={id}>
      {/* <AuthorProfile>: Not Cached (or Short Cached) */}
      <AuthorProfile postId={id} />
    </PostContent>
  );
}

async function PostContent({
  id,
  children,
}: {
  id: number;
  children: React.ReactNode;
}) {
  "use cache";

  const post = await getPost(id);

  return (
    <>
      <h1>{post.title}</h1>
      {/* 📝`ReactNode`はCacheのキーに含まれず、Composableに扱える */}
      {children}
    </>
  );
}
```

### Cacheの課題と解消

- 前述の課題がここまででどうかいしょうされたか整理

## 考察

- 彼らはコミュニティと真摯に向き合った
- 慎重な「抽象化」
- Segment Cacheで同時にRouter Cacheもリアーき

## 感想

- viteのPluginも開発中
- まだまだ発展途上
