---
title: "Cacheの保存先"
---

## 要約

## 背景

- `"use cache"`はNext.jsにCache可能であることを伝えるもの
- Cacheの寿命についてはさまざまなAPIが用意されてる
- Cache保存先も同様に重要な要件で、Next.jsでは保存先を制御するためのAPIが用意されている

## 設計・プラクティス

- Next.jsには、`"use cache"`以外に`"use cache: remote"`や`"use cache: private"`がある
- `"use cache: {name}"`のように、カスタムすることもできるが、代表的なユースケースをフォローするために`"use cache: remote"`や`"use cache: private"`が用意されてる
- `"use cache"`や`"use cache: remote"`はCacheHandlersで制御可能^[VercelなどではStatic Shellの取り扱いが異なるため、必ずしもCacheHandlersが使われるわけではないことに注意]
  - デフォルトだとインメモリに保存されるようになっており、サーバー間共有できない

### `"use cache"`（`"use cache: default"`）

- `"use cache"`の保存先は`default`
- `"use cache"`やStatic Shellは`default`に保存される
  - Vercelにおいては、Static ShellはCDNに保存されrevalidateが可能
  - セルフホスティングの場合はデフォルトだとインメモリに保村されるため、複数プロセスで共有できない
  - 後述の`cacheHandlers.default`で設定は可能だが、まずは `"use cache: remote"`の利用を検討すべき

### `"use cache: remote"`

- 文字通り、リモートにあるCache保存先を選択したい場合に利用することを想定したディレクティブ
- 永続化先としてRedisなどを用意して参照する
- ネットワーク通信は低速なため、`"use cache"`でローカルに保存・参照する方が高速

### `"use cache: private"`(experimental)

- クライアント側に保存されるため、Privateな永続化先を指してることを表すディレクティブ
- 主にユーザー単位データの保存などに利用されることを想定してると考えられる
- experimentalなのは機能開発の途中だからだと思われる

### CacheHandler

- Cacheを複数プロセスで共有するには、`"use cache: remote"`が適切
  - https://github.com/vercel/next.js/issues/85240#issuecomment-3560124078
- セルフホスティングでは`remote`の設定から始めるのが良さそう
- 複数の保存先を設定したい場合、`remote`以外に`session`など独自の命名を追加することも可能
- wattがCacheHandlersを内包してたりする
  - ただし執筆時現在、`default`しか設定されてない
- （OpenNextについても要調査）

## トレードオフ

### プラットフォーム依存なCache制御

- VercelのServerless Functionsでは、Serverlessなので当然ながらインメモリなCacheは扱えない。つまり`"use cache"`を宣言してもCacheされないように見えることがある
  - 仕組みは不明だが、Static Shellは共有できてるらしい
    - https://x.com/108yen___/status/2007466546906714345?s=20
    - [CDNに保存してるらしい](https://github.com/vercel/next.js/discussions/88136#discussioncomment-15423898)が、、明記されてない
- セルフホスティングにおけるマルチプロセスでは、`"use cache: remote"`+`updateTag(tag)`の積極的利用を考えるべき
  - `"use cache"`はプロセス単位のCacheとして、revalidateが不要もしくはCache不整合が許容できるような場合のみ利用するのが良さそう
