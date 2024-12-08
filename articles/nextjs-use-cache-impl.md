---
title: 'Next.js "use cache"の仕組み'
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

`"use cache"`は、Next.jsのdynamicIOという実験的なモードで利用することができる、新たなディレクティブです。本稿は`"use cache"`の仕組みについて、大枠を調査したものになります。

:::message

- 本稿では細かい仕様や内部実装を中心に扱っており、機能自体の説明は省略しています。より詳しく知りたい方は[公式ドキュメント](https://nextjs.org/docs/canary/app)を参照ください。
- Next.jsの実装調査は[564794d](https://github.com/vercel/next.js/tree/564794df56e421d6d4c2575b466a8be3a96dd39a)のコミットを参照しています。最新のコミットでは実装内容が変更されている可能性があります。

:::

## dynamicIO

[dynamicIO](https://nextjs.org/docs/canary/app/api-reference/config/next-config-js/dynamicIO)は2024年のNext Confで発表された、Next.js App Routerにおける新しいコンセプトを実証するための実験的モードです。

https://nextjs.org/blog/our-journey-with-caching

dynamicIOは文字通り、動的なI/O処理に対する振る舞いを大きく変更するものです。具体的には、`fetch()`をはじめとしたデータフェッチは**デフォルトでは実行できなくなります**。データフェッチを扱う際には、`"use cache"`を指定して明示的にキャッシュするか、`<Suspense>`境界内で扱うことで並行実行させるかする必要があります。

`"use cache"`によるキャッシュか、`<Suspense>`によるStreamingの2択になったことで、Next.jsはレスポンスを効率的に送信することができます。また、開発者はデータフェッチをキャッシュするかオンデマンドで扱うかを明示的に選択する必要があります。これにより、dynamicIOは高いパフォーマンスを実現すると同時に、シンプルで明確な開発者体験を提供します。

### App Routerにおけるキャッシュの混乱

dynamicIOは実験的モードですが、いわばNext.jsの未来の1つと考えられます。dynamicIOにより、App Routerの発表から今日まで続くキャッシュに関する多くの混乱を、根本から解決する可能性があります。キャッシュの複雑さはNext.jsに対する最も大きなネガティブ要素だったと言っても過言ではないでしょう。これが大きく変わるとなると、Next.js自体に対する評価も改める必要があるのではないかと筆者は考えています。

### `"use cache"`

さて、このdynamicIOで登場した`"use cache"`ですが、筆者はずっと従来のキャッシュの仕組みの延長線上にあるものだと考えていました。しかし実際に使ってみると、従来とは大きく振る舞いが異なる部分が見受けられ、想像ではなくしっかりと調査する必要性を感じました。

以降は、筆者が`"use cache"`の仕組みについて独自に調査した内容になります。Next.js内部の実装に注目して調査しているため、あまり有用性はないかもしれませんが、興味のある方は参考になれば幸いです。

## `"use cache"`の仕組み

`"use cache"`は大まかに以下のような仕組みに沿って実現されています。

- SWC Pluginで`"use cache"`の対象関数を、Next.js内部定義の`cache()`を通して定義する形に置き換える
- `cache()`は引数にキャッシュのIDや元となる関数などを含み、このIDなどをもとにキャッシュの取得や保存を行う
- キャッシュの永続化はデフォルト^[将来的には[incrementalCacheHandlerPath](https://nextjs.org/docs/app/api-reference/next-config-js/incrementalCacheHandlerPath)同様、カスタマイズ可能になると予想されます。]だとインメモリキャッシュ
- PPRで生成されるキャッシュは`ResumeDataCache`と呼ばれ、上記キャッシュの永続化とは別に存在する

これらについて、実際のコードを見ながら実装を確認します。

### `"use cache"`に対するトランスパイル

Next.js自体はSWCでトランスパイルされ、Server Actionsに対する置き換えの実装はSWC Pluginで実装されています。`"use cache"`の置き換えも、このSWC Pluginに含まれる形で実装されています。

具体的には、`function`内に`"use cache"`があった時には、以下の`if let Directive::UseCache { cache_kind } = directive { ... }`で`"use client"`を判定しています。

:::message
下記は`function`に関する実装ですが、アロー関数にも同様の処理がされます。
:::

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/crates/next-custom-transforms/src/transforms/server_actions.rs#L993-L1032

この処理内で`self.maybe_hoist_and_create_proxy_for_cache_function()`が呼ばれることで、`self.has_cache = true`となります。ここで設定された`self.has_cache`により、以下の分岐に入ります。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/crates/next-custom-transforms/src/transforms/server_actions.rs#L1979-L2001

ここでは対象コードのアウトプットである`new`に対し、コメントにもあるような`import { cache as $$cache__ } from "private-next-rsc-cache-wrapper";`が挿入されるような処理がされています。

さらに`self.maybe_hoist_and_create_proxy_for_cache_function()`の後続処理で、対象の`function`に対し`export var cache_ident = async function() {}`の形に置き換える処理がされます。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/crates/next-custom-transforms/src/transforms/server_actions.rs#L836-L863

上記の`init`で呼ばれる`wrap_cache_expr()`にて、対象の`function`は`$$cache__("name", "id", 0, func)`のような形に置き換えられます。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/crates/next-custom-transforms/src/transforms/server_actions.rs#L2217-L2236

これで、`"use cache"`の対象関数は`private-next-rsc-cache-wrapper`の`cache`関数を介して定義される形になりました。具体的には、以下のようなコードが出力されることになります。

```js
// fixtureより抜粋
// https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/crates/next-custom-transforms/tests/fixture/next-font-with-directive/use-cache/output.js
import { cache as $$cache__ } from "private-next-rsc-cache-wrapper";

export var $$RSC_SERVER_CACHE_0 = $$cache__(
  "default",
  "c0dd5bb6fef67f5ab84327f5164ac2c3111a159337",
  0,
  /*#__TURBOPACK_DISABLE_EXPORT_MERGING__*/ async function Cached({
    children,
  }) {
    return <div className={inter.className}>{children}</div>;
  },
);
```

### `private-next-rsc-cache-wrapper`

上記`private-next-rsc-cache-wrapper`は、webpackのaliasです。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/packages/next/src/build/create-compiler-aliases.ts#L165-L166

上記パスは以下のファイルのbuild結果です。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/packages/next/src/build/webpack/loaders/next-flight-loader/cache-wrapper.ts

`use-cache-wrapper.ts`より`cache()`をそのまま`export`しています。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/packages/next/src/server/use-cache/use-cache-wrapper.ts#L438-L443

この`cache()`関数こそが、`"use cache"`の振る舞いを実装している部分になります。

### `use-cache-wrapper.ts`の`cache()`

`cache()`関数は数百行程度ありますが、ここでは特にキャッシュの永続化の仕組みについて確認します。キャッシュの永続化は以下の2つに分けられているようです。

- `CacheHandler`を経由して行われるもの
- `ResumeDataCache`と呼ばれるもの

#### `CacheHandler`

`CacheHandler`は以下で定義されます。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/packages/next/src/server/use-cache/use-cache-wrapper.ts#L57-L60

設定可能なCacheHandlerのように見えますが、[導入時のPR](https://github.com/vercel/next.js/pull/71505)や筆者が実装を負った限りでは設定できるような手段は見つかりませんでした。そのため、実際には`DefaultCacheHandler`が利用されるものと考えられます。なお、`DefaultCacheHandler`は内省のLRUCacheです。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/packages/next/src/server/lib/cache-handlers/default.ts#L33

#### `ResumeDataCache`

`ResumeDataCache`は少々命名からはわかりづらいですが、PPRにおけるbuild時（prerender）に生成されたキャッシュを引き続きproductionでも利用するためのものです。`cache()`関数では以下で取得されます。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/packages/next/src/server/use-cache/use-cache-wrapper.ts#L552-L557

これらは以下のPRで追加されたようです。

https://github.com/vercel/next.js/pull/72161

prerenderの段階でシードされなかったキャッシュについては、ResumeDataCacheではなくCacheHandlerの方でハンドリングされます。

#### キャッシュの実態はRSC Payload

`CacheHandler`で永続化されるキャッシュも`ResumeDataCache`も、実態はRSC Payloadです。つまり`"use cache"`を適用するとコンポーネントも関数も同様に、RSC Payloadとして内部的に処理されます。そして`cache()`関数では、これらをStreamで扱います。

## 感想
