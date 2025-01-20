---
title: "Container 1stな設計とディレクトリ構成"
---

## 要約

画面の設計はまずContainer Componentsのみで行い、Presentational Componentsは後から追加することを心がけましょう。そうすることでCompositionパターンを早期に適用した設計が可能になり、大きな手戻りを防ぐことができます。

## 背景

[_第1部 データフェッチ_](part_1)ではServer Componentsの設計パターンを、[_第2部 コンポーネント設計_](part_2)ではここまでClient Componentsも含めたコンポーネント全体の設計パターンを解説してきました。ここまで順に読んでいただいた方は、すでに多くの設計パターンを理解されてることと思います。

しかし、これらを「理解してること」と「使いこなせること」は別問題です。

- テストを書けること
- TDDで開発できること

これらに大きな違いがあることと同じく、

- 設計パターンを理解してること
- 設計パターンを駆使して設計できること

これらにも大きな違いがあります。

React Server Componentsでは特に、Compositionパターンを後から適用しようとすると大幅なClient Componentsの設計見直しや書き換えが発生しがちです。こういった手戻りを防ぐためにも、設計の手順はとても重要です。

## 設計・プラクティス

筆者が提案する設計手順は、画面の設計はまずContainer Componentsのみで行い、Presentational Componentsやそこで使うClient Componentsは後から実装する、といういわば**Container 1stな設計手法**です。これは、最初から**Compositionパターンありきで設計する**ことと同義です。

具体的には以下のような手順になります。

1. Container Componentsのツリー構造を書き出す
2. Container Componentsを実装
3. Presentational Components(Shared/Client Components)を実装
4. 2,3を繰り返す

この手順ではServer Componentsツリーをベースに設計することで、最初からCompositionパターンを適用した状態・あるいは適用しやすい状態を目指します。これにより、途中でContainer Componentsが増えたとしても修正範囲が少なく済むというメリットもあります。

:::message
「まずはContainerのツリー構造から設計する」ことが重要なのであって、「最初に決めた設計を守り切る」ことは重要ではありません。実装を進める中でContainerツリーの設計を見直すことも重要です。
:::

### 実装例

よくあるブログ記事の画面実装を例に、Container 1stな設計を実際にやってみましょう。ここでは特に重要なステップ1を詳しく見ていきます。

画面に必要な情報はPost、User、Commentsの3つを仮定し、それぞれに対してContainer Componentsを考えます。

- `<PostContainer postId={postId}>`
- `<UserProfileContainer id={post.userId}>`
- `<CommentsContainer postId={postId}>`

`postId`はURLから取得できますが`userId`はPost情報に含まれているので、`<UserProfileContainer>`は`<PostContainer>`で呼び出される形になります。一方`<CommentsContainer>`は`postId`を元にレンダリングされるので、`<PostContainer>`と並行に呼び出すことが可能です。

これらを加味して、まずはContainer Componentsのツリー構造を`page.tsx`に実際に書き出してみます。各ContainerやPresentational Componentsの実装は後から行うので、ここでは仮実装で構造を設計することに集中しましょう。

```tsx
export default async function Page({
  params,
}: {
  params: Promise<{ postId: string }>;
}) {
  const { postId } = await params;

  return (
    <>
      <PostContainer postId={postId} />
      <CommentsContainer postId={postId} />
    </>
  );
}

async function PostContainer({ postId }: { postId: string }) {
  const post = await getPost(postId);

  return (
    <PostPresentation post={post}>
      <UserProfileContainer id={post.userId} />
    </PostPresentation>
  );
}

// ...
```

ポイントは、`<PostPresentation>`が`children`として`<UserProfileContainer>`を受け取っている点です。この時点でCompositionパターンが適用されているため、`<PostPresentation>`は必要に応じてClient ComponentsにもShared Componentsにもすることができます。

これでステップ1は終了です。以降はステップ2,3を繰り返すので、仮実装にしていた部分を1つづつ実装していきましょう。

1. `<PostContainer>`や`getPost()`の実装
2. `<PostPresentation>`の実装
3. `<CommentsContainer>`の実装
4. ...

本章の主題は設計であり、上記は実装の詳細話になるので、ここでは省略とさせていただきます。

### ディレクトリ構成案

App Routerの規約ファイルはコロケーションを強く意識した設計がなされており、Route Segmentで利用するコンポーネントや関数もできるだけコロケーションすることが推奨されます。

Container/Presentationalパターンにおいて、ページやレイアウトから見える単位はContainer単位です。Presentational Componentsやその他のコンポーネントは、Container Componentsのプライベートな実装詳細に過ぎません。

これらを体現するContainer 1stなディレクトリ設計として、Containerの単位を`_containers`以下でディレクトリ分割することを提案します。このディレクトリ設計では、外部から利用されうるContainer Componentsの公開を`index.tsx`で行うことを想定しています。

```
_containers
├── <Container Name> // e.g. `post-list`, `user-profile`
│  ├── index.tsx // Container Componentsをexport
│  ├── container.tsx
│  ├── presentational.tsx
│  └── ...
└── ...
```

`_containers`のようにディレクトリ名に`_`をつけているのはApp Routerにおけるルーティングディレクトリと明確に区別する[Private Folder](https://nextjs.org/docs/app/building-your-application/routing/colocation#private-folders)の機能を利用するためです。筆者は`_components`、`_lib`のように他のSegment内共通なコンポーネントや関数も`_`をつけたディレクトリにまとめることを好みます。

`app`直下からみた時には以下のような構成になります。

```
app
├── <Segment>
│  ├── page.tsx
│  ├── layout.tsx
│  ├── _containers
│  │  ├── <Container Name>
│  │  │  ├── index.tsx
│  │  │  ├── container.tsx
│  │  │  ├── presentational.tsx
│  │  │  └── ...
│  │  └── ...
│  ├── _components // 汎用的なClient Components
│  ├── _lib // 汎用的な関数など
│  └── ...
└── ...
```

命名やさらに細かい分割の詳細は、プロジェクトごとに適宜修正してもいいと思います。筆者が重要だと思うのはディレクトリをContainer単位で、ファイルをContainer/Presentationalで分割することです。

## トレードオフ

### 広すぎるexport

前述のように、Presentational ComponentsはContainer Componentsの実装詳細と捉えることもできるので、本来Presentational Componentsはプライベート定義として扱うことが好ましいと考えられます。

[ディレクトリ構成例](#ディレクトリ構成案)に基づいた設計の場合、Presentational Componentsは`presentational.tsx`で定義されます。

```
_containers
├── <Container Name> // e.g. `post-list`, `user-profile`
│  ├── index.tsx // Container Componentsをexport
│  ├── container.tsx
│  ├── presentational.tsx
│  └── ...
└── ...
```

上記の構成では`<Container Name>`の外から参照されるモジュールは`index.tsx`のみの想定です。ただ実際には、`presentational.tsx`で定義したコンポーネントもプロジェクトのどこからでも参照することができます。

このような同一ディレクトリにおいてのみ利用することを想定したモジュール分割においては、[eslint-plugin-import-access](https://github.com/uhyo/eslint-plugin-import-access)を利用すると予期せぬ外部からのimportを制限することができます。

上記のようなディレクトリ設計に沿わない場合でも、Presentational ComponentsはContainer Componentsのみが利用しうる**実質的なプライベート定義**として扱うようにしましょう。
