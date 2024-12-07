---
title: 'Next.js "use cache"の仕組み'
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

`"use cache"`は、Next.jsの[dynamicIO](https://nextjs.org/docs/canary/app/api-reference/config/next-config-js/dynamicIO)という実験的なモードで利用することができる、新たなディレクティブです。本稿は`"use cache"`の仕組みについて、筆者の興味の向くままに大枠を調査したものになります。

:::message

- 細かい仕様や内部実装の話が中心となるので、機能の説明は省略している場合があります。機能については[公式ドキュメント](https://nextjs.org/docs/canary/app)をご参照ください。
- 本稿は調査時点の最新のコミットである[564794d](https://github.com/vercel/next.js/tree/564794df56e421d6d4c2575b466a8be3a96dd39a)に基づいています。最新の実装内容とは齟齬がある可能性がありますのでご了承ください。
- 実際に調査していた際のメモは[scrap](https://zenn.dev/akfm/scraps/600da58d6e4717)にあります。

:::

## dynamicIO

dynamicIOは文字通り、動的なI/O処理に対する振る舞いを大きく変更するもので、具体的には`fetch()`をはじめとしたデータフェッチは**デフォルトでは実行できなくなります**。データフェッチを扱う際には、`"use cache"`を指定して明示的にキャッシュするか、`<Suspense>`境界内で扱うことで実行を遅延させるかする必要があります。こうすることで開発者は、明示的にデータフェッチをキャッシュするかオンデマンドで扱うか必ず選択することになります。そしてどちらを選んでも、Streamingを活用した高いパフォーマンスを期待することができます。

dynamicIOは、実験的モードと言われているものの、端的に言えば未来のNext.jsの形の1つです。これによr、App Routerの発表から今日まで続くキャッシュに関する多くの混乱を、根本から解決する可能性があります。キャッシュの複雑さはNext.jsに対する最も大きなネガティブ要素だったと言っても過言ではないでしょう。これが大きく変わるとなると、Next.js自体に対する評価も改める必要があるのではないかと筆者は考えています。

### `"use cache"`

さて、このdynamicIOで登場した`"use cache"`ですが、筆者はずっと従来のキャッシュの仕組みの延長線上にあるものだと考えていました。しかし実際に使ってみると、従来とは大きく振る舞いが異なる部分が見受けられ、これは誤解であったのではないか、しっかり調べるべきでないかと感じました。

以降は筆者が`"use cache"`の仕組みについて、独自に調査した内容になります。Next.js内部の実装に注目して調査しているため、あまり有用性はないかもしれませんが、興味のある方は参考になれば幸いです。

## `"use cache"`の仕組み

`"use cache"`は大まかに以下のような仕組みに沿って実現されています。

- SWC Pluginで`"use cache"`の対象関数を、Next.js独自の`cache()`を通して定義する形に置き換える
- `cache()`は引数にキャッシュのIDや元となる関数などを含み、このIDなどをもとにキャッシュの取得や保存を行う
- キャッシュの永続化はデフォルトだとインメモリキャッシュだが、将来的に設定可能になると予想される
- PPRで生成されるキャッシュは`ResumeDataCache`と呼ばれ、上記キャッシュの永続化とは別に存在する

これらについて、実際のコードを見ながら実装を確認します。

### `"use cache"`に対するトランスパイル

Next.js自体はSWCでトランスパイルされ、Server Actionsに対する置き換えの実装はSWC Pluginで実装されています。`"use cache"`の置き換えも、このSWC Pluginに含まれる形で実装されています。

:::message
Server ActionsのPluginに`"use cache"`の置き換えが含まれるのは違和感がありますが、これは実装初で実験的機能なためかもしれません。
:::

具体的には、`function`内に`"use cache"`があった時には、以下の`if let Directive::UseCache { cache_kind } = directive { ... }`で`"use client"`を判定しています。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/crates/next-custom-transforms/src/transforms/server_actions.rs#L993-L1032

この処理内で`self.maybe_hoist_and_create_proxy_for_cache_function()`が呼ばれることで、`self.has_cache = true`となります。ここで設定された`self.has_cache`により、以下の分岐に入ります。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/crates/next-custom-transforms/src/transforms/server_actions.rs#L1979-L2001

ここでは対象コードのアウトプットである`new`に対し、コメントにもあるような`import { cache as $$cache__ } from "private-next-rsc-cache-wrapper";`が挿入されるような処理がされています。

さらに`self.maybe_hoist_and_create_proxy_for_cache_function()`の後続処理で、対象の`function`に対し`export var cache_ident = async function() {}`の形に置き換える処理がされます。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/crates/next-custom-transforms/src/transforms/server_actions.rs#L836-L863

上記の`init`で呼ばれる`wrap_cache_expr()`にて、対象の`function`は`$$cache__("name", "id", 0, func)`のような形に置き換えられます。

https://github.com/vercel/next.js/blob/564794df56e421d6d4c2575b466a8be3a96dd39a/crates/next-custom-transforms/src/transforms/server_actions.rs#L2217-L2236

これで、`"use cache"`の対象関数は`private-next-rsc-cache-wrapper`の`cache`関数を介して定義される形になりました。上記は`function`に関する実装ですが、アロー関数にも同様の処理がされます。

### `private-next-rsc-cache-wrapper`

### `use-cache-wrapper.ts`
