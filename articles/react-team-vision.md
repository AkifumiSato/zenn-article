---
title: "Reactチームが見てる世界、Reactユーザーが見てる世界"
emoji: "🌏"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: false
---

:::message
この記事は[React Tokyo#2](https://react-tokyo.connpass.com/event/343757/)で公演した[「Reactチームが見てる世界」](https://akifumisato.github.io/slide-of-react-tokyo-202502/1)を、記事として執筆しなおしたものです。
:::

Reactはシンプルなサイトから複雑なアプリケーションまで、非常に幅広く採用されている人気のフレームワークです。OSS化から10年以上の歴史もありながら、昨今も[React Server Components](https://ja.react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023#react-server-components)などの革新的なアイディアを我々に提案し続けています。

一方で、React Server Componentsへの批判的意見や[Boomer Fetching問題](https://github.com/facebook/react/issues/29898)などを見ていると、Reactチームと一部ユーザーの間には意見の相違が見て取れます。この意見の相違は、それぞれが置かれた状況の違いから生じるもの、つまり「見てる世界が違う」ことに起因してると筆者は感じています。

本稿では「Reactチームの見てる世界」を歴史的経緯を踏まえながら考察し、Reactの根本にある思想やコンセプトに対する読者の理解を深めることを目指します。

## 要約

- ReactはMetaの大規模開発を支えるべく開発された技術であり、コンポーネント指向のように**自立分散的**なアーキテクチャこそシンプルでスケーラブルだと考えている
- Reactユーザーの多くは比較的小規模な開発でReactを採用しており、一定規模以下においては**中央集権的**なアーキテクチャが好まれることがある
- Reactチームの自立分散性重視と一部Reactユーザーの中央集権重視な姿勢の違いが、様々な意見の相違を生んでいる

## Reactの誕生

「Reactチームが見てる世界」を深く理解するには、Reactの歴史的経緯を知る必要があります。

Reactが誕生する前、2012年頃のWeb開発は当然ながら今とは大きく状況が異なりました。サーバーサイドではWeb MVC^[MVCはWeb開発以前から存在したアーキテクチャで、Web開発におけるMVCは本来のMVCとは少々異なるそうです。そのため、本稿では区別してWeb MVCと表現しています。]、クライアントサイドではjQueryが良く採用されており、モバイルアプリの台頭もあってAPIとViewの分離ニーズが高まっていました。しかし、jQueryは開発者側で設計する余地が多分に残されていたため、開発体制やアプリケーションのスケールと共に複雑化しやすく、UI開発のスケールは困難でした。

このような状況を改善すべく、`Backbone.js`などのクライアントサイドMVCフレームワーク需要が世間的にも高まっていました。

### `Bolt.js`とReact

Metaにおいても上述の状況は同様で、Meta版`Backbone.js`に相当する`Bolt.js`というクライアントサイドMVCフレームワークが開発されていました。

`Bolt.js`の改善の一環で「UI開発で最も複雑なのは**更新**である」という仮説がうまれ、それを関数型プログラミングのアイディアで解決しようという試みが行われました。具体的にはMVCを削除し、更新に応じて**レンダリング**を行うような変更です。これは当初`FBolt.js`（=Functional `Bolt.js`）と呼ばれてましたが、ここにさらに**JSX**のアイディアを加えたことで、Reactが誕生しました。

誕生当初のReactのコンセプトは、今日においても受け継がれています。

- 関数型指向
- 更新に伴う再レンダリング
- JSX

ここで重要なのは、上述のアイディアを持って**シンプルなフレームワーク**を目指していたことです。最も重視していたのは覚えやすさやパフォーマンスではありません、アプリケーション開発をスケールするシンプルなフレームワークこそ、Reactが目指した姿です。

### Instagramでの採用とReactのOSS化

Reactが最初に採用されたのは、2013年に買収されたばかりのInstagramのWeb UIでした。当時はまだReactは開発途上で、Instagramの開発と並行して改善が進められました。

Instagramで一定の成功が見えた後は、ReactのOSS化が進められました。最初の発表は2013年のJS Conf USでしたが、Reactチームはのちにこの発表を「失敗だった」と表現しています。Web MVCが主流で、HTML・CSS・JavaScriptが技術的関心によって分離されてることこそが良しとされていた時代において、Reactは非常に革新的なアイディアでした。それ故に多くの人々が「うまくいくはずがない」と感じ、反発したようです。実際、レンダリングのアイディアは当初のReactチームメンバーすら「うまくいかないだろう」と感じたメンバーが多かったほどなので、当時の価値観を考えると当然かもしれません。

しかし、「失敗だった」とされるJS Conf USでReactに興味を持った方がいました。後にReactチームに参画する[Sophie Alpert](https://twitter.com/sophiebits)氏です。Sophie Alpert氏はJS Conf US後、Reactに2000行ものコントリビュートを行い、Reactは多くの技術的課題を解消します。多くの改善を経たReactは、改めて同年のJS Conf EUで発表され、これ以降Reactは加速度的に大きな人気を得ていきます。

### Reactの拡大

JS Conf EU以降、ReactはNetflixやAirbnbなど多くの企業で採用されました。関連ライブラリも発展し、Reactのエコシステムと人気は急速に拡大していきます。

その後も、hooksの発明や並行レンダリングなど様々な改善が取り込まれますが、破壊的変更は最小限にとどめられ、Reactの人気は確固たるものになっていきます。

## ReactとGraphQL

React同様、Metaが開発し現在でも広く使われている技術の1つにGraphQLがあります。

Metaでは昔からクライアントサイド・BFF・バックエンドの3層構成を基本としており、クライアントサイドBFF間にREST APIを採用すると発生する以下のような課題を解決すべく開発されたのがGraphQLです。

- 複数エンドポイントからデータ取得するとネットワーク効率が悪い
- Over-fetching: 取得するデータが過剰になる場合がある
- Under-fetching: 取得するデータが不足する場合がある

2013年、Reactが本格的に採用されるとReactとGraphQLを統合する必要が出てきました。これを実現すべく開発が開始されたのが[Relay](https://relay.dev/)です。RelayはGraphQL co-locationを用いて、Reactコンポーネントが必要とするデータを自身で定義できる^[[Thinking in Relay
](https://relay.dev/docs/principles-and-architecture/thinking-in-relay/)]ような自立分散的なアーキテクチャを採用しています。

GraphQLとRelayは2015年にOSS化され、Meta社内ではReact&Relay(GraphQL)の構成がスタンダードとなりました。

### Metaの考えるGraphQL

前述の通りMetaにとってGraphQLはBFFに対する通信、つまりフロントエンドの問題を解決する技術です。BFFバックエンド間の通信は、**Thrift**というRPCが別途開発・採用されています。

一方、Meta以外でGraphQLを採用してるケースでは、バックエンドへの通信プロトコルとして採用されることも多く見られます。

![Metaの考えるGraphQL](/images/react-team-vision/graphql.png)

ここでも、「見てる世界」の違いが見て取れます。GraphQL自体はBFF構成に依存する技術ではないのでバックエンド側でも採用は可能ですが、開発元であるMetaのモチベーションはあくまでフロントエンドの問題を解決することです。

## 自立分散的アーキテクチャ

ReactやGraphQLの歴史的経緯を振り返ると、Metaは一貫して**自立分散的アーキテクチャ**を重視していることがわかります。Reactはコンポーネント指向なフレームワークであり、GraphQLとRelayはコンポーネントが自身で必要なデータを宣言するような設計になっています。

自立分散的でないアーキテクチャとして考えられるのは、**中央集権的アーキテクチャ**です。中央集権的アーキテクチャはWeb MVCやReduxなどが該当すると考えられます。

中央集権的アーキテクチャで考えると、Reactにおけるデータフェッチはトップダウン式になります。

![Top-down Data Fetching](/images/react-team-vision/top-level-data-fetching.png)

一方RelayではGraphQL co-locationを用いて、必要なデータを自身で定義できます。さらに言えば、GraphQLの各Resolverも自律的にデータフェッチを行います。

![GraphQL co-location](/images/react-team-vision/graphql-co-location.png)

### ReduxとMeta

自立分散的アーキテクチャを重視していることを象徴しているのが、ReduxとMetaの関係です。2016年、Redux作者である[Dan Abramov](https://bsky.app/profile/did:plc:fpruhuo22xkm5o7ttr2ktxdo)氏と[Andrew Clark](https://x.com/acdlite)氏がMetaに入社、Reactチームに参画しhooksやReact Fiberなど多くの発展に貢献しています。しかし、彼らが開発したReduxはMeta社内では採用されていません。

Reduxは典型的な中央集権的アーキテクチャです。Metaが抱える大規模開発では、中央集権的アーキテクチャはスケールしないと見なされ、避けられる傾向にあるようです。

## React Server Components

TBW

## Reactチームが見てる世界、Reactユーザーが見てる世界

TBW

## まとめ

TBW

## Memo

- Reactの誕生
  - 2012年のWeb開発
  - 2012年のMeta
  - bolt.js
  - React
  - ReactのOSS化
  - Reactの拡大
- ReactとGraphQL
  - GraphQLの誕生と拡大
  - MetaにとってのGraphQL
  - MetaにとってのReactとGraphQL
- React Server Components
  - 2020年頃、Reactが抱えていた課題
  - React Server Componentsの誕生
  - GraphQLとRSCの自立分散性
- Reactチームが見てる世界、Reactユーザーが見てる世界
  - Reactは大規模開発を支える技術
    - 10万以上のコンポーネントを保有し、修正も自動化してる
    - Metaはデータセンターを世界中に持ってるし、規模が世界有数レベル
  - 大多数のReactユーザーは小規模〜中規模開発でReactを採用してる
- まとめ
  - Reactは大規模開発を支えるシンプルなフレームワークである
  - MetaはReact+GraphQLで自立分散的なアーキテクチャを採用している
  - React Server Componentsもまた自立分散的アーキテクチャであり、正常進化である
