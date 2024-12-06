---
title: 'Next.js "use cache"の仕組み'
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

## 構成

- 序文
  - `"use cache"`は、Next.jsのdynamicIOという実験的なモードで利用することができる、新たなディレクティブです。
  - dynamicIOは`<Suspense>`と`"use cache"`を用いて動的処理とキャッシュのオプトインを可能にします。
  - 具体的には、dynamicとstaticの境界
  - 動的処理はSuspense内で必要に応じて"use cache"を宣言してキャッシュするというシンプルな世界観
  - 従来のdynamic renderingがopt inな世界観から、キャッシュをopt inする形へと変わった
  - 個人的にはこれはとても直感的で、App Routerを再評価する必要がある大きなコンセプトだと思ってる
  - ところで、筆者は"use cache"がこれまでのキャッシュの仕組みの延長にあると思ってた
  - しかし、予想ではなくちゃんと調べて発言しようと思った
  - 本稿はこれを調べた
