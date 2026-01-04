---
title: "高度なCacheと制御"
---

## 要約

## 背景

- `"use cache"`はサーバー側ではStatic Shellの拡張とインメモリなCacheを担う
- インメモリだからサーバー間共有できない
- Cacheしたいがruntime request APIsにアクセスしたいこともある
- このように、Cache要件は多様で1つのディレクティブで全てのケースをフォローはできない

## 設計・プラクティス

- Next.jsには、`"use cache"`以外に`"use cache: remote"`や`"use cache: private"`がある
- `"use cache: {name}"`のように、カスタムすることもできるが、代表的なユースケースをフォローするために`"use cache: remote"`や`"use cache: private"`が用意されてる

### `"use cache: remote"`

TBW

### `"use cache: private"`(experimental)

TBW

### サーバー側Cacheの永続化

- サーバー側Cacheの永続化はCacheHandlersで設定可能
  - Cacheを複数プロセスで共有するには、`"use cache: remote"`が適切
    - https://github.com/vercel/next.js/issues/85240#issuecomment-3560124078
  - セルフホスティングでは`remote`の設定から始めるのが良さそう

## トレードオフ

### プラットフォーム依存なCache制御

- VercelはAWS Lambdaをベースとしているので、インメモリなCacheは扱えない。つまり`"use cache"`を宣言してもCacheされないように見えることがある
  - 仕組みは不明だが、Static Shellは共有できてるらしい
    - https://x.com/108yen___/status/2007466546906714345?s=20
- セルフホスティングにおけるマルチプロセスでは逆に、Static Shellを共有する仕組みを導入しないと`"use cache"`+`updateTag(tag)`が動かないと思われる
  - 複数プロセスでCacheの整合性を担保するには自分でCacheHandlersを設定することが必須となる
