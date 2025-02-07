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

Reactはシンプルなサイトから複雑なアプリケーションまで、非常に幅広く採用されている人気のフレームワークです。OSS化から10年以上の歴史がありながら、昨今も[React Server Components](https://ja.react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023#react-server-components)など革新的なアイディアを我々に提案し続けています。

一方で、React Server Componentsへの批判的意見や[Boomer Fetching問題](https://github.com/facebook/react/issues/29898)などを見ていると、ReactチームとReactユーザーの一部の間には意見の相違が見て取れます。この意見の相違はそれぞれが置かれた状況の違いから生じるもの、つまり「見てる世界が違う」ことに起因してると筆者は感じています。

本稿は「Reactチームの見てる世界」を歴史的経緯を踏まえながら考察し、Reactの根本にある思想やコンセプトに対する読者の理解を深めることを目指します。

## 要約

- ReactはMetaの大規模開発を支えるべく開発された技術であり、**自立分散的アーキテクチャ**こそシンプルでスケーラブルだと、ReactチームやMetaは考えている
- Reactユーザーの多くはMetaより小規模な開発でReactを採用しており、特に一定規模以下においては**中央集権的アーキテクチャ**が好まれることがある
- Reactチームの自立分散性重視と一部Reactユーザーの中央集権重視な姿勢の違いが、様々な意見の相違を生んでいる

## Reactの誕生

「Reactチームが見てる世界」を深く理解するには、Reactの歴史的経緯を知る必要があります。

Reactが誕生する前、2012年頃のWeb開発は当然ながら今とは大きく状況が異なりました。サーバーサイドではWeb MVC^[MVCはWeb開発以前から存在したアーキテクチャで、Web開発におけるMVCは本来のMVCとは少々異なるそうです。そのため、本稿では区別してWeb MVCと表現しています。]、クライアントサイドではjQueryが良く採用されており、モバイルアプリの台頭もあってAPIとViewの分離ニーズが高まっていました。しかし、jQueryは開発者側で設計する余地が多分に残されていたため、開発体制やアプリケーションのスケールと共に複雑化しやすく、UI開発は課題になりがちでした。

このような状況を改善すべく、`Backbone.js`などのクライアントサイドMVC^[Web MVC以前に存在した[元来のMVC](https://ja.wikipedia.org/wiki/Model_View_Controller)は端末側で動作することを想定していたため、クライアントサイドMVCは元来のMVCに近いものだったと考えられます。]フレームワーク需要が世間的にも高まっていました。

### `Bolt.js`とReact

Metaにおいても上述の状況は同様で、Meta版`Backbone.js`に相当する`Bolt.js`というクライアントサイドMVCフレームワークが開発されていました。

`Bolt.js`の改善の一環で「UI開発で最も複雑なのは**UIの更新**である」という仮説がうまれ、それを関数型プログラミングのアイディアで解決しようという試みが行われました。具体的にはMVCではなく、更新に応じて**レンダリング**を行うような変更です。これは当初`FBolt.js`（=Functional `Bolt.js`）と呼ばれてましたが、ここにさらに**JSX**のアイディアを加えたことで、Reactが誕生しました。

https://dionarodrigues.dev/blog/reactjs-behind-the-scenes

誕生当初のReactのコンセプトは、今日においても受け継がれています。

- 関数型指向
- 更新に伴う再レンダリング
- コンポーネント指向
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

Metaでは昔からクライアントサイド・BFF・バックエンドの3層構成を基本としており、クライアントサイド〜BFF間にREST APIを採用すると発生する以下のような課題を抱えていました。

- 複数エンドポイントからデータ取得するとネットワーク効率が悪い
- Over-fetching: 取得するデータが過剰になる場合がある
- Under-fetching: 取得するデータが不足する場合がある

これらを解決すべく開発されたのがGraphQLです。

2013年、Reactが本格的に採用されるとReactとGraphQLを統合する必要が出てきました。これを実現すべく開発されたのが[Relay](https://relay.dev/)です。RelayはGraphQL Colocationを用いて、Reactコンポーネントが必要とするデータを自身で定義できるような自立分散的アーキテクチャを採用しています。

```tsx: author-details.tsx
const authorDetailsFragment = graphql`
  fragment AuthorDetails_author on Author {
    name
    photo {
      url
    }
  }
`;

export default function AuthorDetails({ author }: Props) {
  const data = useFragment(authorDetailsFragment, author);
  // ...
}
```

このことは以下*Thinking in Relay*でも述べられています。

https://relay.dev/docs/principles-and-architecture/thinking-in-relay/

GraphQLとRelayは2015年にOSS化され、Meta社内ではReact&Relay（GraphQL）の構成がスタンダードとなりました。

### Metaの考えるGraphQL

前述の通りMetaにとってGraphQLはBFFに対する通信、つまりフロントエンドの問題を解決する技術です。BFF〜バックエンド間の通信は、**Thrift**というRPCを開発し採用しています。

一方、Meta以外でGraphQLを採用してるケースでは、バックエンドへの通信プロトコルとして採用されることも多く見られます。

![Metaの考えるGraphQL](/images/react-team-vision/graphql.png)

ここでも「見てる世界」の違いが見て取れます。GraphQL自体はBFF構成に依存する技術ではないのでバックエンド側でも採用は可能ですが、開発元であるMetaのモチベーションはあくまで、BFFありきのフロントエンド〜BFF通信の課題を解決することです。そのため、バックエンドとの通信には採用されていません。

## 自立分散的アーキテクチャ

ReactやGraphQLの歴史的経緯を振り返ると、Metaは一貫して**自立分散的アーキテクチャ**を重視していることがわかります。Reactはコンポーネント指向なフレームワークであり、Relayはコンポーネントが自身で必要なデータを宣言するような設計になっています。

自立分散的でないアーキテクチャとして考えられるのは、**中央集権的アーキテクチャ**です。中央集権的アーキテクチャはWeb MVCやReduxなどが該当すると考えられます。

中央集権的アーキテクチャの思考でReactのデータフェッチを考えると、ルートコンポーネントからのバケツリレーを基本としたトップダウン方式になります。

![Top-down Data Fetching](/images/react-team-vision/top-level-data-fetching.png)

一方RelayではGraphQL Colocationを用いて、必要なデータを自身で定義できます。さらに言えば、GraphQLの各Resolverも自律的にデータフェッチを行います。

![GraphQL Colocation](/images/react-team-vision/graphql-co-location.png)

クライアントサイド〜BFFの通信は1つにまとめられていますが、これはRelayがGraphQL Colocationをまとめて1つのQueryに集約していることで成り立っています。

### ReduxとMeta

Metaが自立分散的アーキテクチャを重視していることの象徴の1つが、ReduxとMetaの関係です。2016年、Redux作者である[Dan Abramov氏](https://bsky.app/profile/did:plc:fpruhuo22xkm5o7ttr2ktxdo)と[Andrew Clark氏](https://x.com/acdlite)がMetaに入社Reactチームに参画し、hooksやReact Fiberなど多くの発展に貢献しました。しかし、彼らが開発したReduxはMeta社内では採用されていません。

Reduxは典型的な中央集権的アーキテクチャです。Reduxの作者を迎えてもなお、中央集権的アーキテクチャは採用されませんでした。

## React Server Components

Metaの大規模開発はReact+Relay（GraphQL）により安定した成長を続け、現在でもこれらの技術は多くのプロダクトを支えています。

しかし、当然ながら全く問題がなかったわけではありません。様々なReactアプリケーションの事例や肥大化から、様々な問題が見えてきました。

- バックエンドアクセス
  - 冗長な実装
  - セキュリティ
  - パフォーマンス
- バンドルサイズ
  - 不要なバンドル、冗長なバンドル
  - 最適化コスト
  - 抽象化コスト
- etc...

Reactチームはこれらの問題に対して個別の対処を検討しますが、最終的にはこれらの問題は一貫して「Reactがサーバーを活用できてない」ことに起因してると結論付けられました。この問題に対処すべく新たに設計されたのが、2020年に発表された[React Server Components](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md#motivation)です。

React Server Componentsにより、上記の問題は以下のように解決されました。

- バックエンドフルアクセス
- 0バンドルサイズ（ただし、バンドルサイズは[必ずしも減るわけではない](https://tonyalicea.dev/blog/understanding-react-server-components/)）
- 自動コードスプリッティング
- 0コスト抽象化
- etc...

### React Server Componentsの自立分散性

Server Componentsの登場により、データフェッチもまたコンポーネントにカプセル化することが可能となりました。これは、React Server Componentsもまた自立分散性アーキテクチャを重視していることを示しています。

言い換えると、React Server Componentsが**GraphQLの精神的後継**であるとも言えます。事実、React Server Componentsの最初のRFCは、RelayやGraphQLの発展をリードしてきた[Joe Savona氏](https://twitter.com/en_js)が提案していることからもこのことは見て取れます。

先述のGraphQLの図と、React Server Componentsにおけるデータフェッチを表した図を以下にしまします。

![GraphQL Colocation](/images/react-team-vision/graphql-co-location.png)

![React Server Components](/images/react-team-vision/react-server-components.png)

自律分散性を維持するためにRelayがになっていた層がなくなり、よりシンプルな設計になったと筆者は感じます。

## Reactチームが見てる世界、Reactユーザーが見てる世界

Metaが自立分散的アーキテクチャを重視しているのは、異次元な開発規模が最も大きな理由だと考えられます。Metaは独自のデータセンターを世界中に保有しており、専用のネットワークを持ち、社内のReactコンポーネントは数万以上あるそうです。そして、それら大量のコンポーネントの修正はcode modのようなツールで自動化されてるそうです。

このような異次元な開発規模においては、中央集権的アーキテクチャではなく自立分散的アーキテクチャでこそ最適でスケールできる設計であると、Metaは考えているようです。Metaは実際に10年以上このアーキテクチャで開発し続け、React Server Componentsでもこの姿勢は変わってないことからも、一定の成功を収めているのでしょう。

一方、我々一般的なReactユーザーの多くは、はるかに小規模なアプリケーションでReactを採用してることと思います。Reactが小規模から大規模まで幅広く通用する技術であることは疑いようはないですが、誰しもが自立分散的アーキテクチャを好んでいるわけではなく、中央集権的アーキテクチャこそシンプルだと考える人も一定数います。

| 目線          | 自立分散的アーキテクチャ | 中央集権的アーキテクチャ |
| ------------- | ------------------------ | ------------------------ |
| Reactチーム   | 重視                     | 成り立たない             |
| Reactユーザー | 一定数が重視             | 一定数が重視             |

このように、Reactチームは大規模開発で成り立つ自立分散的アーキテクチャの世界を目指しているのに対し、Reactユーザーの一部は小規模から中規模でのみ成り立つ中央集権的アーキテクチャを好んでいるため、筆者は「見てる世界が違う」と感じています。

### 筆者の見解

最後に誤解なきよう、筆者なりの見解も述べておきます。Reactチームが重視しているのが自立分散的アーキテクチャだからと言って、必ずしもReactにおける**中央集権的アーキテクチャが悪いというわけではない**と、筆者は考えています。むしろ開発者の慣れや状況によって、中央集権の方が好ましいケースは往々にしてあるはずだと考えています。

筆者が重要だと思うのは、我々開発者に**様々な選択肢**があることです。React Server Componentsは自立分散的アーキテクチャを重視して設計されていますが、中央集権的アーキテクチャが実現できないわけではありません。Next.js Pages Routerの`getServerSideProps()`や、Remix v2の`loader()`など、中央集権的アーキテクチャを採用したフレームワークは今後も登場することと思いますし、選択肢が増え議論が活発になることは、技術の発展にも不可欠だとも考えています。

個人的に良くないと思うのは、過剰な批判や無理解です。生産的な議論のためには、相手の状況や思想の理解に努めることが非常に重要だと考えてます。

今後、Next.js App Routerのような自立分散的アーキテクチャと、既存もしくは今後出てくる中央集権的アーキテクチャなフレームワーク、どちらが多くのReactユーザーに受け入れられていくのかは非常に興味深いところです。自立分散性アーキテクチャと中央集権的アーキテクチャ、これらの設計思想を念頭に今後のReact界隈の動向に注目していきましょう。
