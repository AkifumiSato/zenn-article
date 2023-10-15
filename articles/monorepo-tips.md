---
title: "monorepo開発を快適にするツールとtips"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["pnpm", "turborepo", "changesets"]
published: false
---

先日、[`location-state`](https://github.com/recruit-tech/location-state)というパッケージについての記事を公開しました。

https://zenn.dev/akfm/articles/location-state

履歴に基づいて状態を復元できるReact系のライブラリで、現在は[Next.js](https://nextjs.org/)を重点的にサポートしています。このライブラリの構成はcoreとなる部分とNext.js依存な部分を切り離し**scoped package**とし、内部構成もこれに合わせていわゆる**monorepo**構成で開発を行なっています。

- `@location-state/core`: coreとなる部分
- `@location-state/next`: Next.js依存な部分

この記事では実際にmonorepoでパッケージ開発を快適にするために採用したツール・ライブラリを紹介していきます。

:::message
この記事では各ツールの特色や`location-state`で利用している機能を中心に紹介するので、導入方法や詳しい説明はリンク先の公式ドキュメントを参照してみてください。
:::

## 本稿で紹介するツール

本稿で紹介するツールは以下になります。

- `pnpm`: パッケージマネージャーはpnpmがおすすめ、monorepoなら特に
- `turborepo`: remote cacheなしでも十分高速化期待できるので使うべし
- `tsup`: rollup後発的存在、Typescriptをデフォルトサポートが嬉しい
- `changesets`: CHANGELOG生成やnpm publishを自動化できる
- `renovate`: パッケージの自動更新、必須

## pnpm

まずパッケージマネージャーですが、個人的には最近は[pnpm](https://pnpm.io/ja/)一択です。monorepoという面で言うとnpm/yarn/pnpmはどれもworkspaceをサポートしているのですが、pnpmはともかく高速でかつ、Next.js内部でも採用されているなど一定の実績もあります。昨今のyarnはv3移行が滞ってる印象があり、npmはやはり他と比べると遅く感じるという面で、これらのデメリットを補えるpnpmが個人的には好みです。

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

turborepoを使うことで、これらが適切に並列実行・キャッシュされるので、非常に高速に感じられます。逆に使わない場合、これらの依存関係を考慮して手動で実行する必要があり、「えーっと、このパッケージの依存先はxxxだからそっちのテストとbuildも実行しないと...」のように考えることが増えたり、コマンドの実行が漏れてしまったり、逆に必要ないのにbuildしてしまったり、など多くの考慮と手間が発生しやすいです。

ここではturborepoの詳細な仕組みや利用方法については割愛しますが、詳しく知りたい方は以下の記事が参考になるかと思います。

https://zenn.dev/hayato94087/articles/d2956e662202a7

また早く導入して試したいという方は、公式ドキュメントの[Creating a new monorepo](https://turbo.build/repo/docs/getting-started/create-new)や[Add Turborepo to your existing project
](https://turbo.build/repo/docs/getting-started/add-to-project)も参考になると思います。

ちなみにturborepoにはremote cacheという機能もあるのですが、`location-state`では利用してません。（が、それでも非常に高速に感じられています。）

## tsup

package開発ではこれまで[rollup](https://rollupjs.org/)しか使ったことがなかったのですが、`location-state`では[tsup](https://tsup.egoist.dev/)を採用しました。Typescriptデフォルトサポートかつrollup後発ということで良さげということで採用しましたが、実際よかったです。当然ながらtsとの相性もいいし、rollupより学習コストは明らかに低かった気がします。

dtsの生成もフラグでできるのは魅力だったのですが、以下のissueにあるようにCI環境でdtsの生成を待たずに完了してしまうためこの機能は行かせなかったのだけが残念でした。

https://github.com/egoist/tsup/issues/921

ですが結局自前で組むrollupと比較すると手間は変わらないので、パッケージ開発ならtsupはお勧めできるかと思います。

## changesets

パッケージ開発においてリリースノートは変更をユーザーに伝える、重要な要件です。特にmonorepoでは、どのパッケージにどの変更が入ったのかを適切に伝える必要があります。[changesets](https://github.com/changesets/changesets/tree/main)は変更内容を適切にドキュメント管理し、リリースやリリースノートの作成を自動化することができます。

以下に`location-state`での利用している機能について記載します。

### パッケージ修正時の変更管理

パッケージの修正内容は、`changeset add`で変更内容を記載するマークダウンファイルを作成し、適宜修正します。CLIとの対話でおおよそ完成しますが、内容が複雑な場合などは必要に応じてマークダウンを適宜修正します。このマークダウンファイルは、`changeset status`で確認することができます。

例えば`@hoge/fuga`というパッケージのバグ修正でパッチバージョンアップでリリースする場合、以下のようなマークダウンファイルになります。

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

changesetsのgithub actionsを利用するとで、リリースプルリクエストも自動作成されます。パッケージ修正のプルリクエストをマージすると「Version Packages」（名称は変更可能）というプルリクエストが作成されます。

https://github.com/changesets/action#changesets-release-action

このプルリクエストでは、changesetsのマークダウンをまとめてCHANGELOG.mdに出力してくれます。

https://github.com/recruit-tech/location-state/blob/main/packages/location-state-core/CHANGELOG.md

このプルリクエストをマージすると、npmに自動でpublishされリリースノートが作成されます。

## Renovate

[Renovate](https://docs.renovatebot.com/)はパッケージの更新を検知してプルリクエスト〜マージまで自動化することを可能にするbotです。Renovateの運用には信頼度の高いCI実行が前提になりますが、運用できるとパッケージ更新周りの作業をほぼ全自動化できるので非常に楽です。

詳しい導入方法については以下の記事などを参考にすると良いかと思います。

https://zenn.dev/hisamitsu/articles/d41c80ec0ccfb1

`location-state`では1日2回のスケジュールでRenovateがパッケージ更新をチェックし、lint・build・型チェック・単体テスト・結合テストなどを実施し、これらが全て通れば[renovate-approve](https://github.com/apps/renovate-approve)が承認し自動でマージされる運用になっています。

## まとめ

自分の肌感ですが、昨今ではmonorepoでの開発はめずらしくもありません。これはパッケージ開発に限らず、プロダクト開発でも同様です。monorepoでの開発を行う場合、上記のツール・ライブラリはぜひ参考にしてみてください。
