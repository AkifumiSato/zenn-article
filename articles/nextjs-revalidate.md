---
title: "Next.js revalidatePath/revalidateTagの仕組み"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: true
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

## Next.jsのデバッグ

筆者のNext.jsの実装を調査する時のTipsです。興味のある方は参考にしてください。

### ローカル環境でNext.jsをbuildする

forkしたNext.jsのリポジトリを自分の環境に落とし、気になるところに`console.log`を仕込むなど適宜修正buildして調査を行います。buildしたNext.jsをローカルアプリケーションで利用する方法については以下のドキュメントにまとまっています。

https://github.com/vercel/next.js/blob/canary/contributing/core/developing-using-local-app.md

上記手順とは異なりますが、筆者はローカルでNext.jsリポジトリを`pnpm build`した後、以下のように`packages/next`だけ`pnpm create next-app`したアプリケーションで利用する形をとっています。

```shell-session
$ pnpm add {nextjs_git_path}/next.js/packages/next
```

### `NEXT_PRIVATE_DEBUG_CACHE`

`NEXT_PRIVATE_DEBUG_CACHE`という環境変数を設定するとNext.js内部のデバッグ出力が有効になるので、`NEXT_PRIVATE_DEBUG_CACHE=1 next start`のようにして実行するだけでも色々な情報が出力されて内部実装のイメージが掴みやすくなります。

:::message
`next dev`と`next build && next start`では大きく挙動が異なるので、検証やデバッグ時には`next dev`はお勧めしません。
:::

### Next.jsのAPI定義

Next.jsのリポジトリはモノリポ構成になっており、本体である`next`パッケージは`/packages/next`にあります。

https://github.com/vercel/next.js/tree/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next

## `revalidatePath`/`revalidateTag`

上記デバッグ方法を駆使しながら、`revalidatePath`/`revalidateTag`実行時の挙動について調べていきたいと思いますが、その前に公式ドキュメントを確認します。ドキュメントでは以下のような説明がなされています。

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

`revalidatePath("/")`を呼んだのに出力を見るとpathではなく`revalidateTag _N_T_/`となっているのがわかります。また、最後の行を見るとどうやらmanifestを更新してるような出力が見受けられます。この辺りを頭に入れた上で、実際の実装を探っていきましょう。

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

筆者が調べた限り`__NEXT_INCREMENTAL_CACHE_IPC_PORT`などは普通に`next start`しても未定義なので、これらは実質的にVercel専用ロジックではないかと推測されます。なので最後の`return this.cacheHandler?.revalidateTag?.(tag)`がセルフホスティグやローカル環境で実行される処理になります。`this.cacheHandler`は[Custom Next.js Cache Handler](https://nextjs.org/docs/app/api-reference/next-config-js/incrementalCacheHandlerPath)によってカスタマイズすることができますが、前述の通りデフォルトではファイルシステムキャッシュが利用されるので以下の実装が利用されます。

https://github.com/vercel/next.js/blob/efc5ae42a85a4aeb866d02bfbe78999e790a5f15/packages/next/src/server/lib/incremental-cache/file-system-cache.ts#L108-L137

ここでは`this.tagsManifestPath`(`.next/cache/fetch-cache/tags-manifest.json`)に対し、対象タグの`revalidatedAt`を更新してるだけであることがわかります。以下は`tags-manifest.json`の出力例で、`next start`後に`revalidatePath("/cache_debug")`と`revalidateTag("products")`を実行した後の状態です。

```json
{
  "version": 1,
  "items": {
    "_N_T_/cache_debug": { "revalidatedAt": 1711279116900 },
    "products": { "revalidatedAt": 1711279247881 }
  }
}
```

これらのことから、

> これらの関数は呼び出し時のrevalidate情報をサーバー側に保存し、再訪問時にそのrevalidate情報からキャッシュの鮮度を判断してinvalidateを行っている

という筆者の推測通りな実装であるように見受けられました。次に各キャッシュデータのtagとmanifestの比較処理を追ってみます。

## キャッシュデータのtag

キャッシュデータのtagがどこで保存されているのか気になるところですが、闇雲に生成してそうな処理をさがしても非常に膨大な処理を彷徨うことになります。そこで、

> これらの関数は呼び出し時のrevalidate情報をサーバー側に保存し、再訪問時にそのrevalidate情報からキャッシュの鮮度を判断してinvalidateを行っている

という推測の元`.next/cache/fetch-cache/tags-manifest.json`を読み込み利用している箇所を中心に利用箇所を調査しました。その結果、`file-system-cache.ts`の以下の部分がそれらしき処理をしているように見受けられました。

https://github.com/vercel/next.js/blob/e733853cf5ec47bad05ae11dd101f2b6672aa205/packages/next/src/server/lib/incremental-cache/file-system-cache.ts#L265-L291

https://github.com/vercel/next.js/blob/e733853cf5ec47bad05ae11dd101f2b6672aa205/packages/next/src/server/lib/incremental-cache/file-system-cache.ts#L293-L314

それぞれ`data?.value?.kind`という値が`PAGE`か`FETCH`かで分岐しています。`PAGE`はFull Route Cache、`FETCH`はData Cacheを表しています。

Full Route Cacheのtagは`data.value.headers?.[NEXT_CACHE_TAGS_HEADER]`より参照されます。これは内部的には以下の処理で付与されます。

https://github.com/vercel/next.js/blob/canary/packages/next/src/export/routes/app-page.ts#L119-L121

この`fetchTags`は`.next/server`配下の`[pathname].meta`より取得されます。このファイルは`next build`時に生成され、ページに関連するタグ情報を格納しています。

Data Cacheの方では`wasRevalidated`の判定に`tags`と`softTags`を利用しています。fetchで明示的に指定したtagが`tags`、呼び出し元のパス名称に`_N_T_`を付与したtagが`softTags`に含まれています。取得したキャッシュデータである`data`の`lastModified`（ファイルシステムキャッシュの場合、ファイルの更新日時）に対し、それぞれマニフェストに保存されてるrevalidate情報と比較してキャッシュを破棄すべきかどうか検証し、破棄すべきと判断されると`data = undefined`としています。

Data CacheもFull Route Cacheどちらの場合も、`data`が`undefined`だとキャッシュがなかったものとして後続処理に進みます。

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

Next.jsのデバッグや調査も回数を重ねるごとに、以前よりかは小慣れてきたように感じます。この手の大規模な実装の調査スキルは仕事に生きてくる部分もあるし、シンプルに楽しいです。こういった調査を経て`.next`配下やクライアントサイドでの処理の解像度が上がると、バグの調査などが捗ることもメリットと言えそうです。

Next.jsの実装は大規模で筆者が理解してる範囲などごく一部ではあるので、今後もこういった調査を通してよりNext.jsに対する見識を深めていきたいと思います。
