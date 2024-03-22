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

本稿はこれらの記事同様に、`revalidatePath`/`revalidateTag`の仕組みについて筆者が興味のままにNext.jsの実装を調査したことについてまとめたものになります。直接的にApp Routerを利用する開発者に役立つかどうかは分かりませんが、ある程度裏側でこういうことが起きてるんだという参考になれば幸いです。

## 前提条件

App Routerの機能である`revalidatePath`/`revalidateTag`について触れるため、以下の機能について理解してる必要があります。本稿ではこれらの機能について改めて詳解しないので、必要に応じて各ドキュメントをご参照ください。

- [Caching](https://nextjs.org/docs/app/building-your-application/caching)
  - [Router Cache](https://nextjs.org/docs/app/building-your-application/caching#router-cache)
  - [Data Cache](https://nextjs.org/docs/pages/building-your-application/data-fetching)
  - [Full Route Cache](https://nextjs.org/docs/app/building-your-application/caching#full-route-cache)
- [revalidatePath](https://nextjs.org/docs/app/api-reference/functions/revalidatePath)
- [revalidateTag](https://nextjs.org/docs/app/api-reference/functions/revalidateTag)
- [Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)

https://nextjs.org/docs

## Next.jsの調査方法

筆者のNext.jsの実装を調査する時のTipsです。興味のある方は参考にしてください。

### ローカル環境でNext.jsをbuildする

forkしたNext.jsのリポジトリを自分の環境に落とし、気になるところに`console.log`を仕込むなど適宜修正＋buildして調査を行います。buildしたNext.jsをローカルアプリケーションで利用する方法については以下のドキュメントにまとまっています。

https://github.com/vercel/next.js/blob/canary/contributing/core/developing-using-local-app.md

### `NEXT_PRIVATE_DEBUG_CACHE`

`NEXT_PRIVATE_DEBUG_CACHE`という環境変数を設定するとNext.js内部のデバッグ出力が有効になるので、`NEXT_PRIVATE_DEBUG_CACHE=1 next start`のようにして実行するだけでも色々な情報が出力されて内部実装のイメージが掴みやすくなります。

:::message
`next dev`と`next build && next start`では大きく挙動が異なるので、検証やデバッグ時には`next dev`はお勧めしません。
:::

### Next.jsのAPI定義

Next.jsのリポジトリはモノリポ構成になっており、本体である`next`パッケージは`/packages/next`にあります。

https://github.com/vercel/next.js/tree/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next

今回の調査対象である`revalidatePath`などは`next/cache`よりimportするので、`cache.js`から探っていくと良いでしょう。

## `revalidatePath`/`revalidateTag`

`revalidatePath`/`revalidateTag`実行時の挙動について、公式ドキュメントでは以下のような説明がなされています。

> `revalidatePath` only invalidates the cache when the included path is next visited.
> (`revalidatePath`は、含まれるパスへ次に訪問されたときにのみキャッシュを無効にする。)

これらの関数を呼び出した後に、**ページに再訪問することで**キャッシュのinvalidateが行われる、とあります。つまりこれらの関数は呼び出し時のrevalidate情報をサーバー側に保存し、再訪問時にそのrevalidate情報からキャッシュの鮮度を判断してinvalidateを行っている、というような実装であることが推測されます。

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
> (Next.jsでは、PagesとApp Routerのデフォルトのキャッシュハンドラはファイルシステムキャッシュを使用します。)

`revalidatePath("/")`を呼んだのに出力を見ると`revalidateTag _N_T_/`となっているのがわかります。また、最後の行を見るとどうやらmanifestを更新してるような出力が見受けられます。この辺りを頭に入れた上で、実際の実装を探っていきましょう。

### `revalidatePath`/`revalidateTag`の定義から処理を追う

`revalidatePath`/`revalidateTag`の定義は`next/cache`にあります。

https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/cache.js

これ自体jsファイルでbuild済みファイルである`dist`以下を参照しているので、build前の定義である以下を参照します。

https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/web/spec-extension/revalidate.ts#L18-L45

`revalidatePath`と`revalidateTag`の実装の中身を見ていくと、おおよそ大部分の処理が共通化されており、実際には`revalidatePath`は`_N_T_`付のtagをnormalizeして`revalidateTag`を呼び出しているだけであることがわかります。`NEXT_PRIVATE_DEBUG_CACHE=1`にした時の出力結果とも齟齬はありません。

共通化された処理である`revalidate`関数前半部分はチェックやトラッキング用の処理などで、主要な処理は後半部分の以下部分になります。

https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/web/spec-extension/revalidate.ts#L83-L90

`store.pendingRevalidates[tag]`に`incrementalCache.revalidateTag(tag)`の処理自体を格納していることがわかります。この`incrementalCache.revalidateTag(tag)`こそが、revalidate情報を永続化する処理であると推測されます。この処理自体は非同期処理ですがここでは`await`せず、`store`に詰めることで後続処理のServer Actions handler内などで`await`されます。

https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/app-render/action-handler.ts#L622-L625

https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/app-render/action-handler.ts#L112-L124

この`incrementalCache.revalidateTag(tag)`は以下で定義されています。

https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/lib/incremental-cache/index.ts#L278-L295

筆者が調べた限り`__NEXT_INCREMENTAL_CACHE_IPC_PORT`などは普通に`next start`しても未定義なので、これらは実質的にVercel専用ロジックではないかと推測されます。`this.cacheHandler`は[Custom Next.js Cache Handler](https://nextjs.org/docs/app/api-reference/next-config-js/incrementalCacheHandlerPath)によってカスタマイズすることができますが、前述の通りデフォルトではファイルシステムキャッシュが利用されるので以下の実装が利用されます。

https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/lib/incremental-cache/file-system-cache.ts#L108-L137

ここでは`.next/cache/fetch-cache/tags-manifest.json`に最後の`revalidatedAt`を含むよう更新・保存してるだけであることがわかります。他にそれらしいロジックもなく、

> これらの関数は呼び出し時のrevalidate情報をサーバー側に保存し、再訪問時にそのrevalidate情報からキャッシュの鮮度を判断してinvalidateを行っている

という筆者の推測通りな実装であるように見受けられました。次に各キャッシュデータのtagとマニフェストの比較処理を追ってみます。

## キャッシュデータのtag

キャッシュデータのtagがどこで保存されているのか気になるところですが、闇雲に生成してそうな処理をさがしても非常に膨大な処理を彷徨うことになります。そこで、

> これらの関数は呼び出し時のrevalidate情報をサーバー側に保存し、再訪問時にそのrevalidate情報からキャッシュの鮮度を判断してinvalidateを行っている

という推測の元`.next/cache/fetch-cache/tags-manifest.json`を読み込み利用している箇所を中心に利用箇所を調査しました。その結果、`file-system-cache.ts`の以下の部分がそれらしき処理をしているように見受けられました。

https://github.com/vercel/next.js/blob/e733853cf5ec47bad05ae11dd101f2b6672aa205/packages/next/src/server/lib/incremental-cache/file-system-cache.ts#L265-L291

https://github.com/vercel/next.js/blob/e733853cf5ec47bad05ae11dd101f2b6672aa205/packages/next/src/server/lib/incremental-cache/file-system-cache.ts#L293-L314

それぞれ`data?.value?.kind`という値が`PAGE`か`FETCH`かで分岐しています。`PAGE`はFull Route Cache、`FETCH`はData Cacheを表しています。

Full Route Cacheのtagは`data.value.headers?.[NEXT_CACHE_TAGS_HEADER]`より参照されます。これは内部的には以下の処理で付与されます。

https://github.com/vercel/next.js/blob/canary/packages/next/src/export/routes/app-page.ts#L119-L121

この`fetchTags`は`next build`時に生成されます。`.next/server`配下の`[pathname].meta`に関連するtag情報を保存し、実行時にはこれを読み取ることでページに関連する`fetchTags`を実現しています。

Data Cacheの方では`wasRevalidated`の判定に`tags`と`softTags`を利用しています。fetchで明示的に指定したtagが`tags`、呼び出し元のパス名称に`_N_T_`を付与したtagが`softTags`に含まれています。取得したキャッシュデータである`data`の`lastModified`（ファイルシステムキャッシュの場合、ファイルの更新日時）に対し、それぞれマニフェストに保存されてるrevalidate情報と比較してキャッシュを破棄すべきかどうか検証し、破棄すべきと判断されると`data = undefined`としています。

## `revalidate`とRouter Cacheのクリア

`revalidatePath`/`revalidateTags`をServer Actionsで呼びだすと、サーバーからはページのRSC payloadが返され、それを契機にRouter Cacheもパージされます。執筆時点では全てのRouter Cacheがパージされるようになっています。

Server ActionsがページのRSC payloadを返すかは以下`pathWasRevalidated`に基づいて判定されています。

https://github.com/vercel/next.js/blob/e733853cf5ec47bad05ae11dd101f2b6672aa205/packages/next/src/server/app-render/action-handler.ts#L387-L391

RSC payloadはFlight protocolと呼ばれるフォーマットのため、内部的にはFlightと呼ばれることもあり、`skipFlight`となっています。

この`pathWasRevalidated`は以下で設定されています。

https://github.com/vercel/next.js/blob/368e9aa9aedb186ee0dc4e56c89699ece3895cc9/packages/next/src/server/web/spec-extension/revalidate.ts#L89-L90

TODOコメントの通り現状だと無条件に`true`ですが、パスがマッチした場合のみRSCを返す形が理想的ではあります。

### Router Cacheのクリア処理

App Routerのクライアントサイド処理は、`useReducer`で記述された巨大なStateとreducerによって構成されています。Server Actionsがクライアントサイドで実行されると、`server-action`というアクションが発行されdispatchされます。この`server-action`のreducerでページRSC payloadを受け取ったらRouter Cacheのクリアなどを行うようになっています。

https://github.com/vercel/next.js/blob/e733853cf5ec47bad05ae11dd101f2b6672aa205/packages/next/src/client/components/router-reducer/reducers/server-action-reducer.ts#L170-L171

`revalidate`（もしくはcookie操作）しなかった場合、`flightData`は`null`となります。

https://github.com/vercel/next.js/blob/e733853cf5ec47bad05ae11dd101f2b6672aa205/packages/next/src/client/components/router-reducer/reducers/server-action-reducer.ts#L246

後続の上記処理で空のCacheNodeを作成してreducerとして返すことで、Router Cacheをクリアしています。

今回の調査はここまでです。`revalidate`実行時のサーバーサイド〜クライアントサイドの大体の処理イメージは掴めたので、筆者としては満足しました。

## 感想

Next.jsのデバッグや調査も回数を重ねるごとに、以前よりかは小慣れてきたように感じます。この手の大規模な実装の調査スキルは仕事に生きてくる部分もあるし、シンプルに楽しいです。（Next.jsの実装は大規模なので、筆者が理解してる範囲などごく一部ではありますが、、、）

一方こうやって内部実装の理解が進むにつれ、バグ修正などのプルリクエストという形で何かしたくもなるのですが、Next.jsへのプルリクエストは割とスルーされがちなことで有名なので、最近やる気が起きません。こうやって調査するとモチベーション変わってくるかなと思ってたのですが、「どうせスルーされるだろうし」という気持ちは正直あまり変わりませんでした。

しばらくはこうやって内部実装の調査に励んだり、色々機能を試すことでApp Routerと向き合っていこうかなと思います。
