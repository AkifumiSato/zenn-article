---
title: "Next.js App Router revalidatePathで何が起きるのか"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

## 構成

- revalidatePath/revalidateTagの概要
  - 何ができるか
  - どこで呼べるか
- revalidatePathは内部的に何をしているのか
  - 仕様的には「次の訪問時にrevalidateされる」とある
    - どうやって実現しているのだろう？
  - まずはNext.jsのdebugモードを有効にして挙動を確認する
    - `NEXT_PRIVATE_DEBUG_CACHE=1 next start`とすると色々デバッグ出力される
    - この状態で`revalidatePath`を呼ぶと色々出力される
    - どうやらfile system cacheは使っててfetch cacheのin memory storeは使ってないことなどがわかる
    - manifestをアップデートしているようだ
    - あとは実装を追ってみよう
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
