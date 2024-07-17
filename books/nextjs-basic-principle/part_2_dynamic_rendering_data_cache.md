---
title: "dynamic renderingとData Cache"
---

## 要約

Data Cacheを活用して、dynamic rendering時のパフォーマンスを最適化しましょう。

## 背景

- [static renderingとFull Route Cache](part_2_static_rendering_full_route_cache)で述べた通り、App Routerではstatic renderingが推奨されています。
- しかし、ユーザー情報を含むページなどdynamic renderingが必要な場合もあります。
- dynamic renderingではできるだけ早くレンダリングを完了する必要があり、最もボトルネックになるのはデータフェッチ処理です。

## 設計・プラクティス

- dynamic renderingにおいてはData Cacheによってデータの再取得頻度をコントロールすることができます。
- Data Cacheはデフォルトでは永続化され、オプトインでFull Route Cache同様定期的なrevalidateもしくはオンデマンドなrevalidateが可能です。
- これらを適切に設定してData Cacheができるだけヒットするようにすることで、dynamic renderingのパフォーマンスを向上できることがあります。

### オンデマンドrevalidate

- Data Cacheも`revalidatePath()`や`revalidateTag()`でrevalidateできる

1. `revalidatePath()`や`revalidateTag()`
2. 関連するData Cacheをrevalidate
3. 次回訪問時に、Data Cacheが古くなってれば再レンダリング

- これは内部的にData Cacheに関連するRoute情報が付与することによって実現できてる
  - Next.jsの内部的なrevalidateの実装が知りたい方は、下記をご参照ください。
  - https://zenn.dev/akfm/articles/nextjs-revalidate

### 定期的なrevalidate

- `revalidate`を設定することで定期的なrevalidateが可能です
- 前述の理由から、Full Route Cacheも同時にrevalidateされる

### DBアクセスなどのカスタムData Cache

- `unstable_cache()`を使うことで、DBアクセスやGraphQLについてもData Cacheを活用することが可能です。

## トレードオフ

### Data Cacheのオプトアウトとdynamic rendering
