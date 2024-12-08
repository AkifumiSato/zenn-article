---
title: '"use cache"の仕様と実装'
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "swc"]
published: false
---

`"use cache"`はNext.jsにおける新たなディレクティブで、dynamicIOという実験的なモードで利用することができます。本稿は、2024年12月現在における`"use cache"`の仕様や内部実装について解説します。

:::message alert
本稿における調査はNext.jsリポジトリの[564794d](https://github.com/vercel/next.js/tree/564794df56e421d6d4c2575b466a8be3a96dd39a)を参照しています。最新のコミットでは仕様や実装が変更されている可能性があります。
:::

## dynamicIO

[dynamicIO](https://nextjs.org/docs/canary/app/api-reference/config/next-config-js/dynamicIO)は2024年10月のNext Confで発表された、Next.jsにおける新しいコンセプトを実証するための実験的モードです。

https://nextjs.org/blog/our-journey-with-caching

dynamicIOはその名の通り、動的なI/O処理に対する振る舞いを大きく変更するものです。

具体的には、Server Componentsにおける`fetch()`をはじめとしたデータフェッチが**デフォルトでは実行できなくなります**。データフェッチを扱う際には、以下いずれかの対応が必要になります。

- **Dynamic**: 動的にデータフェッチを実行する場合、対象のServer Componentsを`<Suspense>`境界内に配置します。従来同様`<Suspense>`境界内はStreamingで配信されます。
- **Static**: データフェッチがキャッシュ可能な場合やbuild時に実行可能な場合、ファイルや関数の先頭で`"use cache"`宣言します。
- **Partial**: 上記は組み合わせて利用することもできます。

開発者はデータフェッチをいつ実行するべきか選択する必要があるため、従来のようなキャッシュのデフォルト挙動に起因する混乱はおおよそ解消されることが予想されます。一方、ユーザーへのレスポンスをブロックしてデータフェッチを即座に実行する手段がなくなったことで、Next.jsはユーザーに対し効率的にレスポンスを配信することができます。

dynamicIOは高いパフォーマンスを実現すると同時に、シンプルで明確な開発者体験を提供する意欲的なコンセプトであると言えます。

### 私見: キャッシュの混乱とNext.jsへの評価

dynamicIOは現状実験的モードですが、未来のNext.jsのあり方の1つとも考えられます。前述の通り、dynamicIOによってキャッシュのデフォルト挙動にまつわる多くの混乱を根本的に解決する可能性があります。

キャッシュの複雑さはNext.jsに対する最も大きなネガティブ要素だったと言っても過言ではありません。dynamicIOの開発が進むにつれ、**Next.jsに対する評価も大きく改める必要がある**のではないかと筆者は考えています。

## `"use cache"`

`"use cache"`はdynamicIOにおける最も重要なコンセプトです。`"use cache"`はファイルや関数の先頭につけることができ、これによりNext.jsに関数やファイルスコープがキャッシュ可能であることを明示します。

```tsx
// File level
"use cache";

export default async function Page() {
  // ...
}

// Component level
export async function MyComponent() {
  "use cache";
  return <></>;
}

// Function level
export async function getData() {
  "use cache";
  const data = await fetch("/api/data");
  return data;
}
```

`"use cache"`は基本的に引数や参照してるスコープ内の変数などを自動的にキャッシュのキーとして認識しますが、`children`のような一部キーに不適切な値は自動的に除外されます。

より詳細に知りたい方は、以下公式ドキュメントを参照ください。

https://nextjs.org/docs/canary/app/api-reference/directives/use-cache

### キャッシュの永続化

`"use cache"`によるキャッシュは、内部的に以下2つに分類されます。

- オンデマンドキャッシュ: オンデマンド（`next start`以降）で利用されるキャッシュ
- `ResumeDataCache`: [PPR](https://nextjs.org/docs/app/building-your-application/rendering/partial-prerendering)のPrerenderから引き継がれるキャッシュ

オンデマンドキャッシュは、現時点ではシンプルなLRUキャッシュです。内部的には`CacheHandler`という抽象化がされており、将来的には開発者がカスタマイズ可能になることが示唆されています。

`ResumeDataCache`はPPRのPrerenderから引き継がれる特殊なキャッシュで、現時点では`CacheHandler`とは別物になっています。

将来的にこれらは変更されている可能性もありますが、`"use cache"`の振る舞いで悩んだ際には、PPRのために特別なキャッシュが内部的に存在することは注意しておく必要があるかもしれません。

:::message
上記`CacheHandler`は、[`incrementalCacheHandlerPath`](https://nextjs.org/docs/app/api-reference/next-config-js/incrementalCacheHandlerPath)で設定可能なカスタムキャッシュハンドラーとは別物です。
:::

## `"use cache"`の内部実装

ここまではdynamicIOや`"use cache"`について解説してきましたが、以降は`"use cache"`がどう実現されているのか、Next.jsの内部実装について解説します。

`"use cache"`は大まかに以下のような仕組みで実現されています。

1. SWC Pluginで`"use cache"`対象となる関数を、Next.js内部定義の`cache()`を通して定義する形に置き換える
2. `cache()`は引数にキャッシュのIDや元となる関数などを含み、このIDなどをもとにキャッシュの取得や保存を行う

SWC Pluginの置き換え部分から実装を確認していきましょう。

### `"use cache"`に対するトランスパイル

Next.js自体は[SWC](https://swc.rs/)でトランスパイルされ、Server Actions用の`"use server"`に対する置き換えの実装は[SWC Plugin](https://swc.rs/docs/plugin/ecmascript/getting-started)で実装されています。`"use cache"`の置き換えも、このSWC Pluginに含まれる形で実装されています。`"use server"`用のPluginで`"use cache"`に関する処理をしているのはだいぶ違和感がありますが、現状はそうなっているようです。

以下は`function() {}`の先頭に`"use cache"`があった時の処理です。`if let Directive::UseCache { cache_kind } = directive { ... }`で`"use cache"`を判定しています。

:::message
下記は`function() {}`に関する実装ですが、アロー関数（`() => {}`）にも同様の処理がされます。
:::

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/crates/next-custom-transforms/src/transforms/server_actions.rs#L993-L1032

この処理内で`self.maybe_hoist_and_create_proxy_for_cache_function()`が呼ばれることで、`self.has_cache = true`となります。ここで設定された`self.has_cache`により、以下の分岐に入ります。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/crates/next-custom-transforms/src/transforms/server_actions.rs#L1979-L2001

ここでは対象コードのアウトプットである`new`に対し、コメントにもあるような`import { cache as $$cache__ } from "private-next-rsc-cache-wrapper";`が挿入されるような処理がされています。

さらに`self.maybe_hoist_and_create_proxy_for_cache_function()`の後続処理で、対象の`function() {}`に対し`export var {cache_ident} = ...`の形に置き換える処理がされます。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/crates/next-custom-transforms/src/transforms/server_actions.rs#L836-L863

上記の`init`で呼ばれる`wrap_cache_expr()`にて、対象の`function() {}`は`$$cache__("name", "id", 0, function() {})`のような形に置き換えられます。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/crates/next-custom-transforms/src/transforms/server_actions.rs#L2217-L2236

これで、`"use cache"`の対象関数は`private-next-rsc-cache-wrapper`の`cache`関数を介して定義される形になりました。具体的には、以下のようなコードが出力されることになります。

```js
// fixtureから抜粋
// https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/crates/next-custom-transforms/tests/fixture/next-font-with-directive/use-cache/output.js
import { cache as $$cache__ } from "private-next-rsc-cache-wrapper";

// ...

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

上記の置き換えで挿入された`private-next-rsc-cache-wrapper`は、webpackのaliasです。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/packages/next/src/build/create-compiler-aliases.ts#L165-L166

上記パスは以下のファイルのbuild結果で、`use-cache-wrapper.ts`より`cache()`をそのまま`export`しています。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/packages/next/src/build/webpack/loaders/next-flight-loader/cache-wrapper.ts

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/packages/next/src/server/use-cache/use-cache-wrapper.ts#L438-L443

この`cache()`関数こそが、`"use cache"`の振る舞いを実装している部分になります。

### `cache()`

`cache()`関数は数百行程度ありますが、ここでは特にキャッシュの永続化の仕組みについて確認します。キャッシュの永続化は前述の通り以下2つに分けられています。

- `CacheHandler`由来のキャッシュ
- `ResumeDataCache`

#### `CacheHandler`

`CacheHandler`は以下で定義されます。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/packages/next/src/server/use-cache/use-cache-wrapper.ts#L57-L60

設定可能なCacheHandlerのように見えますが、[導入時のPR](https://github.com/vercel/next.js/pull/71505)や筆者が実装を確認した限りでは設定できるような手段は見つかりませんでした。そのため、現状は必ず`DefaultCacheHandler`が利用されるものと考えられます。なお、`DefaultCacheHandler`は内省のLRUCacheです。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/packages/next/src/server/lib/cache-handlers/default.ts#L33

#### `ResumeDataCache`

`ResumeDataCache`は少々命名からはわかりづらいですが、PPRのPrerenderとキャッシュを共有するための仕組みです。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/packages/next/src/server/use-cache/use-cache-wrapper.ts#L552-L557

これらは以下のPRで追加されたようです。

https://github.com/vercel/next.js/pull/72161

Prerenderでシードされなかった`"use cache"`のキャッシュは、`ResumeDataCache`ではなく`CacheHandler`でハンドリングされます。

#### キャッシュの実態

`CacheHandler`で永続化されるキャッシュも`ResumeDataCache`も、実態はRSC Payloadです。つまり`"use cache"`を適用するとコンポーネントも関数も同様に、RSC Payloadとして内部的に処理されます。

`"use cache"`のキャッシュのキーや関数の戻り値はシリアル化可能であるという制約は、内部的にRSC Payloadで扱うことや外部に保存することを考慮しての制約だと考えられます。

おおよそ`"use cache"`を実現するための実装が理解できたので、調査はここまでとしました。

## 感想

これまで筆者はNext.jsのキャッシュの仕組みに関する記事をいくつか執筆してきました。

https://zenn.dev/akfm/articles/next-app-router-navigation

https://zenn.dev/akfm/articles/next-app-router-client-cache

https://zenn.dev/akfm/articles/nextjs-revalidate

上記の記事を執筆してる時にはいつも「複雑さ」を感じていました。しかし今回、`"use cache"`について調査している時に感じたのは「シンプルさ」です。

従来のキャッシュに対するネガティブな意見はデフォルト挙動などが大きかったとは思いますが、それに加え、キャッシュ構造自体があまりに複雑で、利用者側にまでその影響が及んで深い理解を必要としていたことも大きな要因だったのではないかと個人的には考えています。

dynamicIOはシンプルな設計、シンプルな実装の上に成り立っているように感じており、調査を進めるほど期待感が高まりました。

この記事がdynamicIOの理解の参考になれば幸いです。
