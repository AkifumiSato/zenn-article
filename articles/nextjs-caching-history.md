---
title: "Next.js Cache回想"
emoji: "🔄"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: true
---

:::message

- この記事は[JSConf JP 2025で発表した内容](https://jsconf.jp/2025/en/talks/nextjs-caching-re-architecture)を、記事として執筆しなおしたものです。
- この記事は2年前の[JSConf JPで筆者が発表した内容](https://jsconf.jp/2023/talk/akfm-sato-1/)に対するアンサーソングを含みます。

:::

Next.jsは従来より、デフォルトで高いパフォーマンスを実現するフレームワークであることを重視してきました。App Routerにおいても、積極的なCache活用をはじめさまざまなチューニングにより、高いパフォーマンスの実現を目指してきました。一方で、積極的なCache活用や複雑すぎるCacheの影響範囲は、開発者に多くの混乱をもたらしました。

v16で導入された[Cache Components](https://nextjs.org/docs/app/getting-started/cache-components)は、Next.jsのCacheのメンタルモデルを大きく変更するものであり、これまでの多くの課題を解決しうる根本的な**リアーキテクチャ**です。この記事では、Cache Componentsが何を解決しようとしているのか、なぜリアーキテクチャに至ったのかなどの歴史的経緯を解説します。

## Next.jsの歴史

Next.jsには現在、Pages RouterとApp Router2つのRouterが同梱されています^[参考: [App Router and Pages Router](https://nextjs.org/docs#app-router-and-pages-router)]。2016年に発表された初期のNext.jsは現在Pages Routerと呼ばれるものであり、SSGサポートやISRの発明など、静的化によるパフォーマンス最適化を積極的に行ってきました。その後[React Server Components](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md)をサポートした新たなRouterとして、App Routerが開発されました。

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

## Legacy Cache: App Router初期のCache（v13~v14）

App Routerにおいても、Pages Routerで培われた静的化によるパフォーマンス最適化思想は引き継がれています。従来SSGやISRと呼ばれたページの静的化はCacheの1種として位置付けられ、より高度な最適化のために4層のCacheが導入されました。

![Next.jsのCacheの多層化](/images/nextjs-caching-history/cache-layer.png)

以下は[公式の表](https://nextjs.org/docs/app/guides/caching#overview)を翻訳したものです。

| Mechanism               | What              | Where  | Purpose                                    | Duration                                |
| ----------------------- | ----------------- | ------ | ------------------------------------------ | --------------------------------------- |
| **Request Memoization** | APIレスポンスなど | Server | React Component treeにおけるデータの再利用 | リクエストごと                          |
| **Data Cache**          | APIレスポンスなど | Server | ユーザーやデプロイをまたぐデータの再利用   | 永続的 (revalidate可)                   |
| **Full Route Cache**    | HTMLやRSC payload | Server | レンダリングコストの最適化                 | 永続的 (revalidate可)                   |
| **Router Cache**        | RSC Payload       | Client | ナビゲーションごとのリクエスト削減         | ユーザーセッション・時間 (revalidate可) |

これらはそれぞれ最適化の観点が異なるため、理解すれば高度なチューニングや設計が可能です。一方で多層のCacheはそれ自体が複雑なため、バグやドキュメント不足、高い設計難易度など開発者に多くの負担を強いることとなりました。筆者が特に強く課題を感じていた点をいくつか例示します。

### 課題1: 予想困難なデータフェッチ

当時筆者がCache周りで課題に感じていた点を、いくつか例示します。

Webアプリケーション開発において、Cacheできないデータフェッチを実装することは非常に一般的です。App Router初期においては、デフォルトで積極的なCache活用戦略が採用されていたため、データフェッチがCacheされないようにするにはOpt-outが必要でした。しかし、Opt-outの手段も複数あり、影響範囲が広範に及んだため、開発者は多層のCacheと複数のOpt-out手段を考慮する必要がありました。

以下の実装例で考えてみましょう。下記実装例では、リクエストごとにランダムなTodoを取得することを期待して`await getRandomTodo()`を実装しています。

```tsx
export default async function Page() {
  const { todo } = await getRandomTodo();
  // ...
}
```

しかし、このコードの読み手が`await getRandomTodo()`のCache挙動について正しく理解するには、多岐にわたるコードや設定を参照する必要があります。

- Page内で`cookies()`や`headers()`を利用しているかどうか
- `getRandomTodo()`内で呼び出しているであろう`fetch()`で、`cache: "no-store"`や`next: { revalidate: 0 }`がオプション指定されているかどうか
- PageやLayoutでCache挙動に関する[Route Segment Config](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config)が設定されているかどうか

このように、データフェッチをリクエストごとに動的に実行するには広範なコード理解と仕様理解を必要としたため、開発者に大きな負担を強いました。

### 課題2: 複雑なCache設定

前述の通り、Cache挙動はRoute Segment Configで制御することが可能でしたが、これらの設定は高度な理解と広範な影響範囲を伴うため、開発者に大きな負担を強いました。

以下は、Cacheに関係するRoute Segment Configです。

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

これらがLayoutに設定されていると、下層のRoute全てに影響するため、開発者は影響範囲を慎重に設計する必要がありました。

実際には、これらの設定を活用してRoute Segment単位で最適化するより、最上位のLayoutで`export const dynamic = "force-dynamic";`を設定してCacheをデフォルトでOpt-outする方が、一般的なベストプラクティスとなっていたと思います。

### 課題3: 不透明な仕様

筆者の主観ですが、App Router初期におけるCacheは「フレームワーク側が意識するものであって開発者が意識する必要はない」という価値観で設計されていたように感じます。そのため前述の4層のCacheの存在や、各Cacheの詳細な仕様は公式ドキュメントでは明示されておらず、開発者は挙動やコードリーディングから仕様を推測する必要がありました。

このことは2年前のJSConf JPで筆者が発表した内容でも触れています。

https://jsconf.jp/2023/talk/akfm-sato-1/

特に、Router Cacheは30sか5mの間明示的に破棄しなければCacheされるという仕様があり、これが多くの混乱を生みました。

- StaticなRouteは 5m Cacheされる
- DynamicなRouteは 30s Cacheされる

このような仕様はフレームワーク初期におけるバグの多さも相まって、バグなのか仕様なのか判別できず、多くの開発者に混乱と不安をもたらしました。

## Cache Improvement: 議論と改善（v14~v15）

初期のApp RouterにおけるCacheは、動的なWebアプリケーションを開発する開発者にとってともかく悩みの種でした。Next.js開発チームには、SNSやGitHubのissueで多数のフィードバックが寄せられましたが、開発チームの想定とコミュニティの需要には一定認識の乖離が見られました。

このような状況を解消すべく、Next.js開発チームはDiscussionを作成し、そこで開発チームの描いてる世界観とコミュニティの需要をすり合わせ始めました。

https://github.com/vercel/next.js/discussions/54075

「コミュニティが求めてるユースケースは何か？」、「Router Cacheの寿命は設定できれば解決するのか？」「他の解決方法はないのか？」といった議論が重ねられ、Next.js開発チームは改善方針を慎重に検討しました。

### 改善1: `fetch()`のデフォルトCache廃止

`fetch()`がデフォルトでCacheされることについても多くの混乱を招いたため、v15で`fetch()`のデフォルトCacheが廃止されました。

ただし、v15以降も`fetch()`はデフォルトで**Dynamic Renderingにならない**と言う点は、開発者の混乱を解消しきらなかったため、改善はしたが解消はしなかったと筆者は感じています。

```ts
const res = await fetch(`https://...`, {
  // `undefined`(default): Data Cacheは無効、ただしDynamic Renderingには**ならない**
  // `"no-store"`: Data Cacheは無効、Dynamic Renderingを強制
  // `"force-cache"`: Data Cacheは有効
  cache: "force-cache",
});
```

::::details デフォルトの変更に踏み切った背景: PPRの発見

前述の通り、Next.jsはデフォルトで高いパフォーマンスを実現するフレームワークであることを重視してきました。デフォルトでCache活用する戦略の変更は、この価値観に相反する可能性があり、当初Next.js開発チームは慎重な姿勢でした。しかし、Partial Pre-Rendering（PPR）の発見が状況を大きく変えました。

PPRの詳細については、筆者の過去の記事をご参照ください。

https://zenn.dev/akfm/articles/nextjs-partial-pre-rendering

::::

### 改善2: ドキュメントの改善

App Router当初のCacheはともかくドキュメントが不足していたため、仕様を理解するには挙動の観測とコードリーディングが必要でしたが、上記Discussionなどを経て、公式ドキュメントにCache戦略の詳細を解説するページとして[Caching in Next.js](https://nextjs.org/docs/app/guides/caching)が追加されました。これにより、Next.js開発チームが描いてるCache戦略の詳細が明示され、透明性が向上しました。

- 4層のCacheの明記
- 各Cacheの役割
- Static, Dynamic Renderingやこれらの条件になるDynamic APIsの定義

### 改善3: staleTimesオプション

[#課題3: 不透明で不安定なRouter Cache](#課題3-不透明で不安定なrouter-cache)で触れた通り、当初のApp RouterではRouter Cacheの有効期限は固定されており、設定できませんでしたが、これを設定可能にするオプションとしてv14.2で`staleTimes`が追加されました。

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

## Cache Re-Architecture: 根本的な改善（v15~v16）

v14~v15でNext.jsのCacheは着実に改善を重ねました。しかし、多層のCacheや予想困難な影響範囲など、根本的な複雑さは解消されませんでした。

| v13の課題                       | v14~v15での改善                                          | 残った課題                                               |
| ------------------------------- | -------------------------------------------------------- | -------------------------------------------------------- |
| 課題1: 予想困難なデータフェッチ | 改善1: `fetch()`のデフォルトCache廃止                    | 初期よりは改善したが、以前として予想困難なデータフェッチ |
| 課題2: 複雑なCache設定          | -                                                        | 複雑なCache設定の理解と設計難易度                        |
| 課題3: 不透明な仕様             | 改善2: ドキュメントの改善<br>改善3: staleTimesオプション | -                                                        |

これらの解消には破壊的変更を伴うリアーキテクチャが必要だと思われました。そこで打ち出されたのが`"use cache"`というディレクティブでキャッシュを宣言する世界観です。

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

`"use cache"`は2024年のNext Confや公式ブログ[Our Journey with Caching](https://nextjs.org/blog/our-journey-with-caching)で発表されました。発表当時はPPR、`"use cache"`、Dynamic IO^[Dynamic IO: 動的処理を伴うコンポーネントには`<Suspense>`が必須となる世界観]がそれぞれ打ち出されており、最終的な世界観が不明瞭でしたが、2025年に発表されたv16ではこれらを統合する形で[Cache Components](https://nextjs.org/docs/app/getting-started/cache-components)が導入されました。

:::message
Cache Componentsはフラグによって有効化できます。フラグを有効化しなければ従来の世界観を維持することができます。
:::

### `"use cache"`とRSCの世界観

`"use cache"`はRSCで採用されている`"use client"`や`"use server"`と類似した、Next.js独自^[将来的には[vite-plugin-react-use-cache](https://github.com/jacob-ebey/vite-plugin-react-use-cache)など、Next.jsの`"use cache"`相当の機能が他フレームワークで利用可能になる可能性があります。]のディレクティブです。`"use cache"`によってNext.jsは、RSCの世界観を拡張しているとも言い換えられます。

Reactチームの[Dan Abramov氏のブログ](https://overreacted.io/what-does-use-client-do/#two-worlds-two-doors)によると、
RSCの世界観は「_2つの世界、2つのドア_」と見なすことができます。ServerとClient、2つの世界を行き来するためのドアが`"use client"`と`"use server"`です。

![RSC Door](/images/nextjs-caching-history/rsc-door.png =450x)

`"use cache"`も同様に、Serverの向こう側にCacheの世界を定義してServerからCacheの世界へのドアとみなすことができます。

![Cache Components](/images/nextjs-caching-history/cache-door.png)

また、`"use cache"`が宣言されたコンポーネントは、`"use client"`同様Compositionパターンが適用可能です。

```tsx
/**
 * example:
 * <PostContent id={id}> // Cached
 *   <AuthorProfile postId={id} /> // Not Cached
 * </PostContent>
 */
async function PostContent({
  id,
  children,
}: {
  id: number;
  // 📝`ReactNode`のようなSerializableできないものは、Cacheのキーに含まれない=Composable
  children: React.ReactNode;
}) {
  "use cache";
  const post = await getPost(id);

  return (
    <>
      <h1>{post.title}</h1>
      {children}
    </>
  );
}
```

RSCはじめ、昨今ReactチームはComposableな設計思想を重視しています。`"use cache"`も同様に、Composableな設計を踏襲している点も、RSCの世界観と整合性が取れている点です。

### 解消された課題

`"use cache"`によってCacheは宣言的になったため、従来の

- 前述の課題がここまででどうかいしょうされたか整理

### Cache Componentsとパフォーマンス

- 課題解消のネックだったパフォーマンス懸念について

## 考察

- 彼らはコミュニティと真摯に向き合った
- 慎重な「抽象化」
- Segment Cacheで同時にRouter Cacheもリアーき

## 感想

- viteのPluginも開発中
- まだまだ発展途上
