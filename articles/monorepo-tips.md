---
title: "monorepo開発を快適にするツール選定"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["pnpm", "turborepo", "changesets", "tsup", "renovate"]
published: true
---

先日、[`location-state`](https://github.com/recruit-tech/location-state)というパッケージについての記事を公開しました。

https://zenn.dev/akfm/articles/location-state

履歴に基づいて状態を復元できるReact系ライブラリで、現在は[Next.js](https://nextjs.org/)を重点的にサポートしています。このライブラリの構成はcoreとなる部分とNext.js依存な部分を切り離し**scoped package**とし、内部構成もこれに合わせていわゆる**monorepo**構成で開発を行なっています。

- `@location-state/core`: coreとなる部分
- `@location-state/next`: Next.js依存な部分

この記事では、monorepoのパッケージ開発を快適にするために実際に採用したツールを紹介していきます。本稿で紹介するツールは以下になります。

- [pnpm](#pnpm)
- [turborepo](#turborepo)
- [tsup](#tsup)
- [changesets](#changesets)
- [Renovate](#Renovate)

:::message
この記事では各ツールの特色や`location-state`で利用している機能を中心に紹介するので、導入方法や詳しい導入方法や説明は各公式ドキュメントを参照してください。
:::

## pnpm

まずパッケージマネージャーですが、個人的には最近は[pnpm](https://pnpm.io/ja/)一択です。monorepoという面で言うとnpm/yarn/pnpmはどれもworkspaceをサポートしているのですが、pnpmはともかく高速でかつ、Next.js内部でも採用されているなど一定の実績もあります。昨今のyarnはv2,3移行が滞ってる印象があり、npmはやはり他と比べると遅く感じるという面で、これらのデメリットを補えるpnpmが個人的には好みです。

また、pnpm workspaceはワークスペースプロトコル`workspace:`に対応しています。これはyarn v2以降で実装されている機能で、パッケージ間の参照を容易にするものです。この機能はyarn v1やnpmにはないので、monorepo開発時にpnpmを採用するメリットの1つと言えます。

例えば`@location-state/next`では、`@location-state/core`や内部パッケージの`configs`や`eslint-config-custom`の参照を`workspace:*`で行なっています。

https://github.com/recruit-tech/location-state/blob/cf96ed297a056bf1de3e68bd2e9c9559690086d1/packages/location-state-next/package.json#L32-L34

## turborepo

[turborepo](https://turbo.build/repo)はmonorepoのbuildをはじめとしたコマンド実行の並列化・キャッシュで高速化を図るツールです。monorepoはパッケージ間で依存関係が発生するために、パッケージのbuildやテストが影響し合うことがあります。turborepoは、monorepoの解析・設定に基づいて依存関係を考慮した並列実行を行なうことで、長くなりがちなmonorepoのbuildやテストを高速化することができます。

例をあげると、`@location-state/next`のbuildをturborepoなしに行おうとすると、以下の順番で行う必要があります。

1. `@location-state/core`の`d.ts`生成
1. `@location-state/next`の`d.ts`生成
1. 内部パッケージ`test-utils`のbuild
1. `@location-state/core`のbuild
1. `@location-state/next`のbuild

turborepoを使わない場合はこれらの依存関係を考慮して手動で実行する必要があり、「えーっと、このパッケージの依存先はxxxだからそっちのテストとbuildも実行しないと...」のように考えることが増えたり、コマンドの実行が漏れてしまったり、逆に必要ないのにbuildしてしまったり、など多くの考慮と手間が発生しやすいです。turborepoを使うことで、これらが適切に並列実行・キャッシュされるので、非常に高速に感じられます。

turborepoの仕組みや利用方法について詳しく知りたい方は、以下の記事が参考になるかと思います。

https://zenn.dev/hayato94087/articles/d2956e662202a7

また早く導入して試したいという方は、公式ドキュメントの[Creating a new monorepo](https://turbo.build/repo/docs/getting-started/create-new)や[Add Turborepo to your existing project
](https://turbo.build/repo/docs/getting-started/add-to-project)も参考になると思います。

ちなみにturborepoには[remote cache](https://turbo.build/repo/docs/core-concepts/remote-caching)という機能もありますが、`location-state`では利用してません。**それでも、非常に高速に感じられています**。

## tsup

package開発ではこれまで[rollup](https://rollupjs.org/)しか使ったことがなかったのですが、`location-state`では[tsup](https://tsup.egoist.dev/)を採用しました。Typescriptデフォルトサポートかつrollup後発で良さげ、ということで採用しましたが実際よかったです。筆者がrollupを使ったのが結構前なので、昔のrollupと比較するとの感想になってはしまいますが、Typescriptとの相性もいいのでrollupより学習コストは低く感じました。

[dtsの生成もフラグでできる](https://tsup.egoist.dev/#generate-declaration-file)のは魅力だったのですが、以下のissueにあるようにCI環境でdtsの生成を待たずに完了してしまうためこの機能は生かせなかったのだけが残念でした。

https://github.com/egoist/tsup/issues/921

ですが自前で組むrollupと比較すると結局手間は変わらないので、パッケージ開発ならtsupはお勧めできるかと思います。

## changesets

パッケージ開発においてリリースノートは変更をユーザーに伝える、重要な要件です。特にmonorepoでは、リリース時にどのパッケージにどんな変更が入ったのかを適切に伝える必要があります。[changesets](https://github.com/changesets/changesets/tree/main)は変更内容を適切にドキュメント管理し、リリースやCHANGELOG、リリースノートの作成を自動化することができます。

changesetsは他と比べると少々使い勝手のイメージがつきにくいと思うので、機能ごとに紹介します。

### パッケージ修正時の変更管理

パッケージの修正時に`changeset add`で変更内容を記載するマークダウンファイルを作成し、適宜修正します。CLIとの対話でおおよそ完成しますが、内容が複雑な場合などは必要に応じてマークダウンを適宜することができます。

例えば`@hoge/fuga`というパッケージのバグ修正でpatchリリースしたい場合、以下のようなマークダウンファイルになります。

```md
---
"@hoge/fuga": patch
---

Fix bug xxx.
```

### 変更対象パッケージの判定とchangeset不在の検知

上記修正のプルリクエスト作成時に`changeset add`し忘れたまま修正プルリクを出した場合、以下のようにchangesetsのbotを利用すると自動で以下のように指摘コメントしてくれます。

![changesetsのコメント](https://user-images.githubusercontent.com/11481355/66183943-dc418680-e6bd-11e9-998d-e43f90a974bd.png)

差分にchangesetsのマークダウンが見つかった場合、変更されるパッケージのバージョンなどをコメントしてくれます。

### リリースプルリクエストの作成と自動publish

[changesetsのgithub actions](https://github.com/changesets/action#changesets-release-action)を利用するとで、リリースプルリクエストも自動作成されます。パッケージ修正のプルリクエストをマージすると「**Version Packages**」（名称は変更可能）というプルリクエストが作成されます。

このプルリクエストでは、changesetsのマークダウンをまとめてCHANGELOG.mdに出力してくれます。以下は`location-state`のCHANGELOG.mdです。

https://github.com/recruit-tech/location-state/blob/main/packages/location-state-core/CHANGELOG.md

プルリクエストをマージすると、npmに自動でpublishされリリースノートが作成されます。

https://github.com/recruit-tech/location-state/releases/tag/%40location-state%2Fnext%401.0.1

## Renovate

[Renovate](https://docs.renovatebot.com/)はパッケージの更新を検知してプルリクエスト〜マージまで自動化することを可能にするbotです。Renovateの運用には信頼度の高いCI実行（パッケージのは快適変更を検知するテストやlint）が前提になりますが、運用できるとパッケージ更新周りの作業をほぼ全自動化できるので非常に楽です。

詳しい導入方法については以下の記事などを参考にすると良いかと思います。

https://zenn.dev/hisamitsu/articles/d41c80ec0ccfb1

`location-state`では1日2回のスケジュールでRenovateがパッケージ更新をチェックし、lint・build・型チェック・単体テスト・結合テストなどを実施し、これらが全て通れば[renovate-approve](https://github.com/apps/renovate-approve)が承認し**自動でマージされる運用**になっています。

## まとめ

昨今ではmonorepoでの開発はめずらしくもありません。これはパッケージ開発に限らず、プロダクト開発でも同様です。monorepoでの開発を行う場合、上記のツール・ライブラリはぜひ参考にしてみてください。
