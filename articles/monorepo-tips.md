---
title: "monorepo開発を快適にするツールとtips"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["pnpm", "turborepo", "changesets"]
published: false
---

最近、`location-state`というパッケージを開発しています。履歴に基づいて状態を復元できる、主にNext.jsをサポート対象としたReact系のライブラリです。このライブラリの構成はcoreとNext.js依存な部分を切り離すなど、いわゆるmonorepo構成で開発を行なっています。この記事では実際にmonorepoでパッケージ開発を快適にするために採用したツール・ライブラリを紹介していきます。

## pnpm

まずパッケージ管理ツールですが、個人的には最近はpnpm一択です。monorepoという面で言うとworkspaceはどれもサポートしているのですが、pnpmはともかく高速でかつ、Next.js内部でも採用されているなど一定の実績もあります。昨今はyarnはv3が個人的には先行きに不安を感じる部分があり、npmはやはり他と比べると遅く感じるという面で、これらのデメリットを補えるpnpmが個人的には好みです。

また、pnpm workspaceでは`workspace`プロトコルに対応しており、これが非常に便利です。

TBW

- pnpm
  - パッケージ管理ツールとしては個人的には一択
    - workspaceはどれもサポートしてるが
    - yarnはv3が先行き不安
    - npmは遅い
    - pnpmは速い、採用事例も多い
  - pnpm workspaceでは`workspace`プロトコルに対応しており、これが非常に便利
    - 開発中にパッケージ間でのバージョン管理をあまり気にしなくて良くなる（と思う、他わからんけど）
- turborepo
  - monorepoのbuildを並列化・キャッシュしてくれるツール
  - 目玉機能だと思ってたremote cacheは今回使ってないが、それでも非常に高速に感じられる
- tsup
  - package開発ではrollupしか使ったことなかったけど、今はこれがよさげらしい、ということで採用
  - 当然ながらtsとの相性もいいし、rollupより学習コストは明らかに低かった気がする
  - dtsの生成もフラグでできるのは魅力だが、CI環境でdtsの生成を待たずに完了してしまうためこの機能は行かせなかったのが残念
- changesets
  - 変更履歴をマークダウンで管理できるツール
  - これだけ読むと正直あまりそそられなかったのだが、公式が用意してくれてるgithub actionsやbotが非常に便利
  - github actionsでpublishをトリガーするPRを作成してくれるので、リリースが簡単にできる
  - mainブランチにマージするごとに、`@next`を自動でpublishしてくれる機能も持つ
  - botがマークダウンの書き忘れを促しつつ、影響するパッケージを教えてくれる
  - リリースノートはchangesetがまとめてくれる
- renovate
  - パッケージのアップデートを自動でPRを作成してくれるbot
  - CI環境をちゃんと作っておけば安心してPRからマージまで任せられる
- まとめ
  - `pnpm`: パッケージ管理はこれでいいのでは
  - `turborepo`: remote cacheなしでも十分高速化期待できるので使うべし
  - `tsup`: 外部向けpackageがなくとも、内部packageのbuildに使うと思うのでおすすめ
  - `changesets`: 外部向けpackageの開発ならおすすめ
  - `renovate`: CI整えて自動マージまで組むと幸福度高い

