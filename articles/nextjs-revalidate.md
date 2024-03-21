---
title: "Next.js revalidatePathの仕組み"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

以前、Next.jsの遷移の実装やRouter Cacheの実装について筆者が調べたことを記事にしました。

https://zenn.dev/akfm/articles/next-app-router-navigation
https://zenn.dev/akfm/articles/next-app-router-client-cache

本稿はこれらの記事同様に、`revalidatePath`/`revalidateTag`の仕組みについて筆者が興味のままにNext.jsの実装を調査したことについてまとめたいと思います。

直接的にApp Routerを利用する開発者に役立つかどうかは分かりませんが、ある程度裏側でこういうことが起きてるんだという参考になれば幸いです。

## 前提条件

App Routerの機能である`revalidatePath`/`revalidateTag`について触れるため、以下の機能について理解してる必要があります。本稿ではこれらの機能について改めて詳解しないので、必要に応じて各ドキュメントをご参照ください。

- [Caching](https://nextjs.org/docs/app/building-your-application/caching)
  - [Data Cache](https://nextjs.org/docs/pages/building-your-application/data-fetching)
  - [Full Route Cache](https://nextjs.org/docs/app/building-your-application/caching#full-route-cache)
- [revalidatePath](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)
- [revalidateTag](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)
- [Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)

https://nextjs.org/docs

## Next.jsの調査方法

筆者のNext.jsの実装を調査する時のTipsです。興味のある方は参考にしてください。

### ローカル環境でNext.jsをbuildする

forkしたNext.jsのリポジトリを自分の環境に落とし、気になるところに`console.log`を仕込んだり適宜修正しながらbuildして調査を行います。ローカル環境でbuildしたNext.jsをアプリケーションで利用する方法については以下のドキュメントにまとまっています。

https://github.com/vercel/next.js/blob/canary/contributing/core/developing-using-local-app.md

### `NEXT_PRIVATE_DEBUG_CACHE`

`NEXT_PRIVATE_DEBUG_CACHE`という環境変数を設定するとNext.js内部のデバッグ出力が有効になるので、`NEXT_PRIVATE_DEBUG_CACHE=1 next start`のようにして実行するだけでも色々な情報が出力されて内部実装のイメージが掴みやすくなります。

:::message
`next dev`と`next build && next start`では大きく挙動が異なるので、検証やデバッグ時には`next dev`はお勧めしません。
:::

### Next.jsのAPI定義

Next.jsのリポジトリはものリポ構成になっており、本体である`next`パッケージは以下にあります。

https://github.com/vercel/next.js/tree/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next

今回の調査対象である`revalidatePath`などは`next/cache`よりimportするので、`cache.js`から探っていくと良いでしょう。

## `revalidatePath`/`revalidateTag`

`revalidatePath`/`revalidateTag`実行時の挙動について、公式ドキュメントでは以下のような説明がなされています。

> `revalidatePath` only invalidates the cache when the included path is next visited.
> `revalidatePath`は、含まれるパスが次に訪問されたときにのみキャッシュを無効にする。(Deepl)

これらの関数を呼び出した後に、**ページに再訪問することで**キャッシュのinvalidateが行われる、とあります。つまりこれらの関数は呼び出されたrevalidate情報を永続化し、再訪問時にそのrevalidate情報と比較することでキャッシュのinvalidateの要否を判断している、というような実装であることが推測されます。

### `revalidatePath`のデバッグ出力

実際に`NEXT_PRIVATE_DEBUG_CACHE=1`を設定し、以下のような簡易的なサンプルで実行して`revalidatePath`のデバッグ出力を確認してみます。

```tsx
// app/page.tsx
import { revalidatePath } from "next/cache";

export default async function Page() {
  async function revalidate() {
    "use server";

    revalidatePath("/");
  }

  const res = await fetch("https://dummyjson.com/products/1", {
    next: { tags: ["products"] },
  }).then((res) => res.json());

  return (
    <>
      <h1>Hello, Next.js!</h1>
      <code>
        <pre>{JSON.stringify(res, null, 2)}</pre>
      </code>
      <form action={revalidate}>
        <button type="submit">revalidate</button>
      </form>
    </>
  );
}

export const dynamic = "force-dynamic";
```

```log
using filesystem cache handler
not using memory store for fetch cache
revalidateTag _N_T_/
Updated tags manifest { version: 1, items: { '_N_T_/': { revalidatedAt: 1710837210692 } } }
```

出力にもある通り、Next.jsのキャッシュ永続化先はデフォルトだとファイルシステムキャッシュとなっていることがわかります。これは[公式ドキュメント](https://nextjs.org/docs/app/api-reference/next-config-js/incrementalCacheHandlerPath)にもあるので想定通りです。

> In Next.js, the default cache handler for the Pages and App Router uses the filesystem cache.
> Next.jsでは、PagesとApp Routerのデフォルトのキャッシュハンドラはファイルシステムキャッシュを使用します。

`revalidatePath("/")`を呼んだのに出力を見ると`revalidateTag _N_T_/`となっているのがわかります。また、最後の行を見るとどうやらmanifestを更新してるような出力が見受けられます。この辺りを頭に入れた上で、実際の実装を探っていきましょう。

### `revalidatePath`/`revalidateTag`の定義から処理を追う

`revalidatePath`/`revalidateTag`の定義は`next/cache`にあります。

https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/cache.js

これ自体jsファイルでbuild済みのファイルを参照しているので、参照先は直接Github上だと参照できません。build前の定義は以下のtsファイルになります。

https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/web/spec-extension/revalidate.ts#L18-L45

`revalidatePath`と`revalidateTag`の実装の中身を見ていくと、おおよそ大部分の処理が共通化されており、実際には`revalidatePath`は`_N_T_`付のtagをnormalizeして`revalidateTag`を呼び出しているだけであることがわかります。

`revalidate`関数も前半部分でチェックやトラッキング用の処理などしていますが、メインの処理は後半部分の以下の部分です。

https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/web/spec-extension/revalidate.ts#L83-L90

`store.pendingRevalidates[tag]`に`incrementalCache.revalidateTag(tag)`の処理自体を格納していることがわかります。この`incrementalCache.revalidateTag(tag)`こそが、revalidate情報を永続化する処理であると推測されます。この処理自体は非同期処理ですがここでは`await`せず、`store`に詰めることで後続処理のServer Actions handler内などで`await`されます。

https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/app-render/action-handler.ts#L622-L625

https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/app-render/action-handler.ts#L112-L124

この`incrementalCache.revalidateTag(tag)`は以下で定義されています。

https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/lib/incremental-cache/index.ts#L278-L295

筆者が調べた限り`__NEXT_INCREMENTAL_CACHE_IPC_PORT`などは普通に`next start`しても未定義なので、これらはVercel専用ロジックじゃないかと推測しています。`this.cacheHandler`は[Custom Next.js Cache Handler](https://nextjs.org/docs/app/api-reference/next-config-js/incrementalCacheHandlerPath)によってカスタマイズすることができますが、前述の通りデフォルトではファイルシステムキャッシュが利用されるので以下の実装が利用されます。

https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/lib/incremental-cache/file-system-cache.ts#L108-L137

ここでは`.next/cache/fetch-cache/tags-manifest.json`に最後の`revalidatedAt`を含むよう更新・保存してるだけであることがわかります。他にそれらしいロジックもなく、

> これらの関数は呼び出されたrevalidate情報を永続化し、再訪問時にそのrevalidate情報と比較することでキャッシュのinvalidateの要否を判断している

という筆者の推測通りな実装であるように見受けられました。しかし「再訪問時にそのrevalidate情報と比較することで」という部分が当然これらの処理に含まれていないので、次に各キャッシュデータのtagが内部的にどう紐付けられているのか確認してみましょう。

## キャッシュデータのtag

TBW

## 構成

- 各データのtagはどこに保存されるのか
  - FS Cacheの`get`時にキャッシュが古いかどうかを検証してる
    - Full Route Cache: https://github.com/vercel/next.js/blob/e733853cf5ec47bad05ae11dd101f2b6672aa205/packages/next/src/server/lib/incremental-cache/file-system-cache.ts#L276-L282
    - Data Cache: https://github.com/vercel/next.js/blob/e733853cf5ec47bad05ae11dd101f2b6672aa205/packages/next/src/server/lib/incremental-cache/file-system-cache.ts#L296-L308
      - tagsに明示的に指定したタグ、softTagsに`_N_T_`付のタグが入ってる
  - https://github.com/vercel/next.js/blob/canary/packages/next/src/export/routes/app-page.ts#L119-L121
    - build時にここでtagを`.meta`ファイルに書き込んでる
- revalidateをSAで呼ぶとRouter Cacheがクリアされる
  - https://github.com/vercel/next.js/blob/e733853cf5ec47bad05ae11dd101f2b6672aa205/packages/next/src/server/app-render/action-handler.ts#L387-L391
    - ここでレスポンスのRSCを生成してる
    - skipFlightにrevalidateの有無を渡し、不要なら空のFlightを作ってるっぽい
- `server-action`のreducerでRSCを受け取ったらRouter Cacheのクリアなどを行う
  - https://github.com/vercel/next.js/blob/e733853cf5ec47bad05ae11dd101f2b6672aa205/packages/next/src/client/components/router-reducer/reducers/server-action-reducer.ts#L210
    - ここがページのRSCが帰ってきたときの処理
  - https://github.com/vercel/next.js/blob/e733853cf5ec47bad05ae11dd101f2b6672aa205/packages/next/src/client/components/router-reducer/reducers/server-action-reducer.ts#L246
    - ここで空のRouter Cacheを生成してる
