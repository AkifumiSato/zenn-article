---
title: "Next.js revalidatePathの仕組み"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

以前App Routerの遷移やRouter Cacheについて、内部仕様・実装の解説記事を書いたところ、結構反響がありました。

https://zenn.dev/akfm/articles/next-app-router-navigation
https://zenn.dev/akfm/articles/next-app-router-client-cache

今回はずっと筆者が気になっていた、`revalidatePath`/`revalidateTag`の仕組みについてNext.jsの実装を調査してみたので、上記記事同様に内部実装や仕様についてのまとめたいと思います。

直接的にApp Routerを利用する開発者に役立つかどうかは分かりませんが、ある程度裏側でこういうことが起きてるんだという参考になれば幸いです。

## 対象読者の前提条件

App Routerの機能である`revalidatePath`/`revalidateTag`について触れるため、以下の機能について理解してる必要があります。本稿ではこれらの機能について改めて詳解しないので、必要に応じて各ドキュメントをご参照ください。

https://nextjs.org/docs

- [Caching](https://nextjs.org/docs/app/building-your-application/caching)
  - [Data Cache](https://nextjs.org/docs/pages/building-your-application/data-fetching)
  - [Full Route Cache](https://nextjs.org/docs/app/building-your-application/caching#full-route-cache)
- [revalidatePath](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)
- [revalidateTag](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)
- [Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)

## `revalidatePath`/`revalidateTag`は内部的に何をしてるか

`revalidatePath`/`revalidateTag`は

> `revalidatePath` only invalidates the cache when the included path is next visited.

と公式ドキュメントにあるように、これらの関数を呼んだ後にページに再訪問することでキャッシュのinvalidateが行われます。ではこれらの関数は内部的に何をやってるのでしょう。この手の調査を行う時筆者は、forkしたNext.jsのリポジトリをbuildしてデバッグしながら調査を行います。ローカル環境でのbuildは以下を参照ください。

https://github.com/vercel/next.js/blob/canary/contributing/core/developing-using-local-app.md

また、非公式ですが`NEXT_PRIVATE_DEBUG_CACHE=1 next start`のように`NEXT_PRIVATE_DEBUG_CACHE`という環境変数を設定すると、Next.js内部のデバッグ出力が有効になるので、これだけでも色々な情報が出力されてイメージが掴みやすくなります。

:::message
`next dev`と`next build && next start`では大きく挙動が異なるので、検証やデバッグ時には`next dev`はお勧めしません。
:::

### `NEXT_PRIVATE_DEBUG_CACHE`を有効にしてデバッグ出力を確認する

上記環境変数を設定した状態で`revalidatePath`を実行すると、色々気になる出力が得られます。以下のような簡易的なサンプルで実行してみます。

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

出力にもある通り、Next.jsのキャッシュ永続化先はデフォルトだとファイルシステムキャッシュとなっています。`revalidatePath("/")`を呼んだのに出力を見ると`revalidateTag _N_T_/`となっているのは、内部的に`revalidatePath`は`N_T/`というprefixをつけただけでほとんど`revalidateTag`と同様の処理となっているためです。

最後の行を見ると、どうやらmanifestを更新してるような出力が見受けられます。この辺りを頭に入れた上で、実装を探っていきましょう。

### `revalidatePath`の定義から処理を追う

TBW

## 構成

- revalidatePathは内部的に何をしているのか
  - `next/cache`から追ってく
    - https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/cache.d.ts
    - https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/web/spec-extension/revalidate.ts
      - 基本的な処理自体はrevalidatePathとrevalidateTagで共通化されており、内部的にはtag名称を`_N_T_`付でnormalizeしてるだけ
    - `store.pendingRevalidates[tag]`にincrementalCache.revalidateTag()の処理自体を格納してる
      - ↓Server Actionsのhandler内などでawaitされる
      - https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/app-render/action-handler.ts#L622-L625
    - https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/web/spec-extension/revalidate.ts#L84
    - https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/lib/incremental-cache/index.ts#L278
      - IPCはデフォルトではないのでL.294の`return this.cacheHandler?.revalidateTag?.(tag)`
      - `CacheHandler`はデフォルトではFileSystemCache
        - FetchCacheと言うのもあるけど、`x-vercel-sc-host`というヘッダー見て有効判定してるからおそらくVercel専用
    - https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/lib/incremental-cache/file-system-cache.ts#L108
      - `.next/cache/fetch-cache/tags-manifest.json`を更新して保存してるだけ
        - manifestに最後の`revalidatedAt`を保存して、以降のリクエストでキャッシュとmanifestの`revalidatedAt`を比較して、キャッシュ利用有無を判断してるのである
        - ということで次はデータ側のtagがどう紐づけられてるか確認する
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

## 全体のポイント

- `NEXT_PRIVATE_DEBUG_CACHE=1`をつけると色々わかるよ
- revalidatePath/revalidateTagは基本同じ処理で、manifestを更新するのみである
- デフォルトではファイルシステムキャッシュが利用されるが、Vercel専用っぽいやつがあったり、開発者が任意のカスタムハンドラー設定できるよ
