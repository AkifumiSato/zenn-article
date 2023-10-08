---
title: "monorepo開発を快適にするツールとtips"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["pnpm", "turborepo", "changesets"]
published: false
---

先日、`location-state`というパッケージ開発についての記事を公開しました。

https://zenn.dev/akfm/articles/location-state

履歴に基づいて状態を復元できるReact系のライブラリで、現在はNext.jsを重点的にサポートしています。このライブラリの構成はcoreとなる部分とNext.js依存な部分を切り離しscoped packageとする、いわゆるmonorepo構成で開発を行なっています。

- `@location-state/core`: coreとなる部分
- `@location-state/next`: Next.js依存な部分

この記事では実際にmonorepoでパッケージ開発を快適にするために採用したツール・ライブラリを紹介していきます。

## pnpm

まずパッケージ管理ツールですが、個人的には最近は[pnpm](https://pnpm.io/ja/)一択です。monorepoという面で言うとworkspaceはどれもサポートしているのですが、pnpmはともかく高速でかつ、Next.js内部でも採用されているなど一定の実績もあります。昨今のyarnはv3移行が滞ってる印象があり、npmはやはり他と比べると遅く感じるという面で、これらのデメリットを補えるpnpmが個人的には好みです。

また、pnpm workspaceはワークスペースプロトコル`workspace:`に対応しています。これはyarn v2で実装された機能で、パッケージ間の参照を容易にするものです。

例えば`@location-state/next`では、`@location-state/core`や内部パッケージの`configs`や`eslint-config-custom`の参照を`workspace:*`で行なっています。

https://github.com/recruit-tech/location-state/blob/cf96ed297a056bf1de3e68bd2e9c9559690086d1/packages/location-state-next/package.json#L32-L34

この機能があることも、monorepo開発時にpnpmを採用するメリットの1つと言えます。

## turborepo

[turborepo](https://turbo.build/repo)はmonorepoのbuildをはじめとしたコマンド実行の並列化・キャッシュで高速化を図るツールです。monorepo間の依存関係を自動解決しつつ、設定に基づいてコマンド実行の依存関係も考慮してくれるので、修正時にコマンド実行漏れなどが防げるのにキャッシュヒットで高速に感じられるツールです。

例をあげると、`@location-state/next`のbuildをturborepoなしに行おうとすると、以下の順番で行う必要があります。

1. `@location-state/core`の`d.ts`生成
1. `@location-state/next`の`d.ts`生成
1. 内部パッケージ`test-utils`のbuild
1. `@location-state/core`のbuild
1. `@location-state/next`のbuild

turborepoを使うことで、これらが適切に並列実行・キャッシュされるので、非常に高速に感じられます。

ここでは詳細な仕組みや利用方法については割愛しますが、詳しく知りたい方は以下の記事が参考になるかと思います。

https://zenn.dev/hayato94087/articles/d2956e662202a7

また早く導入して試したいという方は、公式ドキュメントの[Creating a new monorepo](https://turbo.build/repo/docs/getting-started/create-new)や[Add Turborepo to your existing project
](https://turbo.build/repo/docs/getting-started/add-to-project)も参考になると思います。

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
