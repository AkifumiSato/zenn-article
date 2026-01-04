---
title: '"use cache"とCache制御関数'
---

## 要約

## 背景

- 従来のNext.jsのCacheはデフォルトで有効で、Cacheできないような処理を行うことで暗黙的にOpt-outされるような設計だった
- このような設計をわかりづらいと感じる人は多かったため、Cache ComponentsではデフォルトでCacheされず、`"use cache"`によってOpt-inできるような設計に変更された
- より詳しい歴史的経緯については、以下の記事を参照

https://zenn.dev/akfm/articles/nextjs-caching-history

## 設計・プラクティス

- `"use cache"`はComponentや関数がCache可能であることを宣言するもの
- Static Shellを拡張したり、リクエスト間でCacheを共有したりできる。ただし...
  - https://github.com/vercel/next.js/issues/85240#issuecomment-3560124078
    - `"use cache"`はClientやServerのインメモリなCacheと考えておくのが良さそう
    - マルチプロセスのセルフホスティングにおいてはCache不整合の原因になるし、VercelのCacheHandlerは永続化しないよう設定されてるので、「リクエスト間でCacheを共有」するなら`"use cache: remote"`を使いましょう（次章にて解説）

### `"use cache"`

TBW

### `cacheLife(profile)`

TBW

### `cacheTag(...tags)`

TBW

### `updateTag(tag)`

TBW

### `revalidateTag(tag)`

TBW

## トレードオフ

### 自動で算出されるCacheのkey

TBW
