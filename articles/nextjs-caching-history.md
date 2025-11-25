---
title: "Next.js Cache回想"
emoji: "📖"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: true
---

:::message
この記事は[JSConf JP 2025で発表した内容](https://jsconf.jp/2025/en/talks/nextjs-caching-re-architecture)を、記事として執筆しなおしたものです。
:::

Next.jsは従来より、デフォルトで高いパフォーマンスを実現するフレームワークであることを重視してきました。App RouterにおいてもSSR・Streaming・Cacheなど様々な最適化により、高いパフォーマンスの実現を目指してきました。しかし一方で、積極的なCache活用や複雑すぎるCacheの影響範囲は、開発者に多くの混乱をもたらしました。

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

Pages Routerは現在でも利用可能ですが新規の機能開発などは行われておらず、現在はApp Routerが主流^[参考: [Vercelのテックリードのツイート](https://x.com/timneutkens/status/1982905913741644089)]となっています。

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

これらのCacheを使いこなすことで高いパフォーマンスの実現が期待できる一方、多層のCacheはそれ自体が高い認知負荷を伴い、そこにさまざまな課題が重なったことで、結果的に開発者に大きな負担を強いることとなりました。

当時筆者がCache周りで課題に感じていた点を、いくつか例示します。

### 課題1: 予想困難なデータフェッチ

Webアプリケーション開発において、Cacheできないデータフェッチを実装することは非常に一般的です。App Router初期においては、デフォルトで積極的なCache活用戦略が採用されていたため、データフェッチがCacheされないようにするにはOpt-outが必要でしたが、Opt-outの手段は複数あり、影響範囲も広範に及んだため、開発者に高い認知負荷を強いました。

下記の実装例では、リクエストごとにランダムなTodoを取得することを期待して`await getRandomTodo()`を実装しています。

```tsx
export default async function Page() {
  const { todo } = await getRandomTodo();
  // ...
}
```

しかし、このコードの読み手が`await getRandomTodo()`のCache挙動について正しく理解するには、多岐にわたるコードや設定を参照する必要があります。具体的には、`getRandomTodo()`の内部実装と呼び出し元の実装、その双方を確認しなければOpt-outされるかどうかを正しく判断できませんでした。

- `getRandomTodo()`内で呼び出しているであろう`fetch()`で、`cache: "no-store"`や`next: { revalidate: 0 }`がオプション指定されているかどうか
- `getRandomTodo()`の内または外で、`cookies()`や`headers()`などを利用しているかどうか
- Pageや祖先のLayoutでCacheに関する[Route Segment Config](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config)が設定されているかどうか

このように、サーバー側CacheであるFull Route CacheやData CacheがOpt-outされるかどうかを実装から判断するには、広範なコード理解とNext.jsのCache仕様理解を必要としました。

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

ダッシュボードのような非常に動的なアプリケーションの開発においてはCacheを一律Opt-outすることも多く、筆者の主観ですが、これらの設定は「駆使する」より「避ける」方がベストプラクティスとなっていたと思います。

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

このDiscussionを通してNext.js開発チームは改善方針を慎重に検討し、v14~15にかけて様々な改善を実施しました。以下は、v14~15で行われた改善の一部です。

### 改善1: `fetch()`のデフォルトCache廃止

v15では、`fetch()`によるData Cacheがデフォルトで無効化されました。これは破壊的変更でしたが、従来`fetch()`がデフォルトでCacheすることについては多くの混乱が見られたため、筆者はとても大きな改善だったと捉えています。

一方で、Full Route CacheをOpt-outしてDynamic Renderingするには、引き続き`fetch()`に`cache: "no-store"`を指定する必要がありました。この点において、`fetch()`のデフォルトの振る舞いは引き続き開発者に混乱を招いたため、状況の改善こそ見られたものの、**根本的な課題の解消には至らなかった**と筆者は捉えています。

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

これらの解消には破壊的変更を伴うリアーキテクチャが必要だと思われました。そこで打ち出されたのが`"use cache"`で明示的にキャッシュをOpt-inするという世界観です。

`"use cache"`は2024年のNext Confや公式ブログ[Our Journey with Caching](https://nextjs.org/blog/our-journey-with-caching)で発表されました。発表当時はPPR・`"use cache"`・Dynamic IO^[Dynamic IO: 動的処理を伴うコンポーネントには`<Suspense>`が必須となる世界観]がそれぞれ打ち出されており、最終的な世界観が不明瞭でしたが、2025年に発表されたv16ではこれらを統合する形で[Cache Components](https://nextjs.org/docs/app/getting-started/cache-components)フラグが導入されました。

### `"use cache"`

`"use cache"`は関数やコンポーネントの先頭、もしくはファイルの先頭で宣言することができます。

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

前述の`getRandomTodo()`の場合、従来は広範なコード理解とNext.jsのCache仕様理解を必要としましたが、`"use cache"`がある世界では`getRandomTodo()`からデータフェッチまでの間に`"use cache"`があるかどうか調べるだけで良くなります。

```tsx
async function getRandomTodo() {
  "use cache";

  // await fetch("...")
}
```

`"use cache"`により、暗黙的なCacheから明示的なCacheへ、グローバルな設定からローカルな宣言へと大きく設計志向が変更されました。

### `"use cache"`とRSCの世界観

`"use client"`や`"use server"`はRSCの仕様として定義されているディレクティブですが、`"use cache"`はNext.js独自^[将来的には[vite-plugin-react-use-cache](https://github.com/jacob-ebey/vite-plugin-react-use-cache)など、Next.jsの`"use cache"`相当の機能が他フレームワークで利用可能になる可能性があります。]のディレクティブです。言い換えると、Next.jsは`"use cache"`によって、RSCの世界観を拡張していると言えます。

ReactチームのDan Abramov氏は[ブログ記事](https://overreacted.io/what-does-use-client-do/#two-worlds-two-doors)にて、RSCの世界観を「_2つの世界、2つのドア_」と説明しています。ServerとClient、2つの世界を行き来するためのドアが`"use client"`と`"use server"`です。

![RSC Door](/images/nextjs-caching-history/rsc-door.png =450x)

`"use cache"`も同様に、Serverの世界から**Cacheの世界**へのドアを定義してるものと看做すことができます。

![Cache Components](/images/nextjs-caching-history/cache-door.png)

また、`"use cache"`が宣言されたコンポーネントがComposable^[Reactチームは昨今、Composable=合成可能な設計思想を重視しています。]であることも、RSCの設計思想と一致していると言えます。

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

このように、`"use cache"`にはRSCの世界観との類似性を見出すことができます。

### Cacheの課題と解消まとめ

v13からv16までで、Cacheに関する課題がどう解消されたか整理します。

| v13の課題                       | v14~v15での改善                                          | v16（Cache Components）での改善                   |
| ------------------------------- | -------------------------------------------------------- | ------------------------------------------------- |
| 課題1: 予想困難なデータフェッチ | 改善1: `fetch()`のデフォルトCache廃止                    | `"use cache"`による明示的なCache宣言              |
| 課題2: 複雑なCache設定          | -                                                        | `"use cache"`によるOpt-in型の設計と設定変数の廃止 |
| 課題3: 不透明な仕様             | 改善2: ドキュメントの改善<br>改善3: staleTimesオプション | -                                                 |

また、これらの過程を経てRequest Memoizationを除く全てのCacheが**Opt-out型からOpt-in型へと変更**された点も、非常に大きな改善点です。Next.js開発チームは段階を経つつ、最終的には根本的な課題解消まで踏み込む形で大きく改善を実施しました。

### 新たな課題

Next.jsのCacheはここまでで明らかに改善されたと感じる一方、もう課題がないわけではありません。Cache Componentsの安定性向上と機能追加はこれからだと思われますし、コミュニティ側も`"use cache"`について理解を深めるのにまだ時間がかかると思われます。

個人的にはセルフホスティングでNext.jsを利用することが多いため、Cache Componentsで追加された新たな[cacheHandler**s**](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheHandlers)との向き合い方も重要な課題です。従来の[cacheHandler](https://nextjs.org/docs/app/api-reference/config/next-config-js/incrementalCacheHandlerPath)（従来の設定には`s`がない）における[@neshca/cache-handler](https://caching-tools.github.io/next-shared-cache/)のように、3rd partyのライブラリが一定の抽象化を担ってくれれば、導入ハードルが一気に下がると思われます。

また、`"use cache"`のより発展的なディレクティブとして、新たなディレクティブを導入する動きも見受けられます。[`"use cache: private"`](https://nextjs.org/docs/app/api-reference/directives/use-cache-private)と[`"use cache: remote"`](https://nextjs.org/docs/app/api-reference/directives/use-cache-remote)は公式ドキュメントでそれぞれ説明を確認することができますが、まだ開発チームから大きなアナウンスはありません。このように派生系のディレクティブが増えていくことがコミュニティに受け入れられるのかどうかも含め、動向を注視する必要があります。

## 私見

最後に、ここまで述べたCacheの変遷について筆者なりに私見を述べます。

### 考察

`"use cache"`の設計をリードしたのは[Sebastian Markbåge](https://ja.react.dev/community/team#sebastian-markb%C3%A5ge)氏です。彼はReact開発チームとNext.js開発チームを兼任するメンバーであり、ReactやNext.jsのビジョンに強く貢献してきた人物です。ReactのhooksやReact Fiberのビジョンはまさに、彼が大きく貢献した部分です。

以下はJSConf EU 2014での彼の発言です。

> "It's much easier to recover from no abstraction than the wrong abstraction."
>
> 「間違った抽象化から回復するよりも、抽象化がない状態から回復する方がずっと簡単だ」

このように彼は、抽象化の導入には慎重な検討が必要というスタンスをReact初期から貫いています。今回のCacheの変遷についても、彼のこのスタンスが強く感じられます。当初のCacheは開発者が強く意識しなくていいようにしたいという意図も起因して、非常に具体的な設定やAPIに依存する設計となっていました。しかし、その後のコミュニティからのフィードバックやPPRの発見などを経て、`"use cache"`という抽象化にいたったと言えます。

筆者が思うに、当初から`"use cache"`という抽象化を見出すことは不可能だったことでしょう。`"use cache"`は彼やNext.js開発チームの慎重な検討と、数多のフィードバックの積み重ねがあってこその成果だと思います。

### 感想

`"use cache"`の登場により、`"use client"`や`"use server"`を含めた「ディレクティブによる抽象化」に対する是非の議論が、昨今再燃してるように筆者は感じています。SNSなどを観察してると肯定的な意見も多いようですが、批判的な意見も目立ちます。これに関する筆者の意見としては、既存エコシステムにおける実現性やJavaScriptの仕様における制約などの観点から、「理想解ではないが現実解」だったのではないかと考えています。実際、`"use client"`の仕様策定に関する[当時のディスカッション](https://github.com/reactjs/rfcs/pull/189)を読み直してみても、筆者にはなかった案や観点で多くの議論がなされており、彼らが筆者よりはるかに広い観点でディレクティブの導入について検証していたことを感じます。

今回の`"use cache"`についても同様に、広い観点と多くの議論を経たからこその「優れた抽象化」を筆者は感じています。あれだけ複雑だったCacheをRSCの世界観と繋げて再設計したこと、それでいてRSCの世界観を一切損なってないことに筆者は非常に驚愕しましたし、長い時間をかけ攻撃的な批判にも負けず、コミュニティの課題を解消すべく真摯に向き合い続けてくれたNext.js開発チームの姿勢には、感謝の念を抱かずにはいられません。

Cache ComponentsでPPR・Dynamic IO・`"use cache"`を統合した1つの世界観が示されたことで、Next.jsのCacheは1つ大きな節目を迎えつつあるのを筆者は感じています。しかし、残念ながらこれでNext.jsのCacheのストーリーは終わりではありません。`"use cache: private"`や`"use cache: remote"`のように、Next.jsはCacheをさらに進化させていくことでしょう。これらがコミュニティに受け入れられるのかどうか、また議論の末変更されるのかなど、引き続き状況を注視していく必要があります。

ただ、ここまでのNext.js開発チームの仕事ぶりは素晴らしいものだったと筆者は思います。そしてこれからも、時に間違いつつもまた慎重に検討を重ねて、Next.js開発チームは「優れた抽象化」を見つけていくことでしょう。これからも筆者は、Next.jsのCacheのストーリーを1開発者として追っていきたいと思います。
