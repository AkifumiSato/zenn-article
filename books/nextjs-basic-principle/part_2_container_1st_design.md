---
title: "Container 1stな設計とディレクトリ構成"
---

## 要約

画面の設計はまずContainer Componentsのみでまず行い、Presentational Componentsは後から追加することを心がけましょう。

## 背景

[_第1部_](part_1)ではServer Componentsの設計パターンを、[第2部](part_2)ではここまでClient Componentsも含めたコンポーネント全体の設計パターンを解説してきました。ここまで順に読んでいただいた方はすでに多くの設計パターンを理解されてることと思います。

しかし、これらを理解してることと使いこなせることは別問題です。

- テストを書けること
- TDDで開発できること

これらに大きな違いがあることと同じく、

- 設計パターンを理解してること
- 設計パターンを駆使して設計できること

これらにも大きな違いがあります。

React Server Componentsでは特に、Compositionパターンを後から適用しようとしたりすると大幅なClient Componentsの設計見直しや書き換えが発生しがちです。こういった手戻りを防ぐためにも、設計の手順はとても重要です。

## 設計・プラクティス

筆者が提案する設計手順は、画面の設計はまずContainer Componentsのみで行い、Presentational Componentsやそこで使うClient Componentsは後から実装する、といういわば**Container 1stな設計手法**です。これは、最初からCompositionパターンありきで設計することと同義です。

具体的には以下のような手順になります。

1. Container Componentsのツリー構造を書き出す
2. Container Componentsを実装
3. Presentational Componentsを実装
4. 必要に応じてClient Componentsを実装
5. 2-4を繰り返す

この手順は最初からCompositionパターンありきで設計してるため、途中でContainer Componentsが増えたとしても修正範囲が少ないというメリットがあります。

:::message
「まずはContainerのツリー構造から設計する」ことが重要なのであって、「最初に決めた設計を守り切る」ことは重要ではありません。実装を進める中でContainerツリーの設計を見直すことも重要です。
:::

### 実装例

よくあるブログ記事の画面実装を例に、Container 1stな設計を実際にやってみましょう。ここでは特に重要なステップ1を詳しく見ていきます。

画面に必要な情報はPost、User、Commentsの3つを仮定し、それぞれに対してContainer Componentsを考えます。

- `<PostContainer postId={postId}>`
- `<UserProfileContainer id={post.userId}>`
- `<CommentsContainer postId={postId}>`

`postId`はURLから取得できますが`userId`はPost情報に含まれているので、`<UserProfileContainer>`は`<PostContainer>`で呼び出される形になります。一方`<CommentsContainer>`は`<PostContainer>`と並列に呼び出すことが可能です。

これらを加味してまずはContainer Componentsのツリー構造を`page.tsx`に実際に書き出してみます。各ContainerやPresentational Componentsの実装は後から行うので、ここでは仮実装で構造を設計することに集中しましょう。

```tsx
export default async function Page({
  params: { postId },
}: {
  params: { postId: string };
}) {
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

これでステップ1は終了です。以降はステップ2-4の通り、仮実装にしていた部分を1つづつ実装していきましょう。

1. `<PostContainer>`や`getPost()`の実装
2. `<PostPresentation>`の実装
3. `<CommentsContainer>`の実装
4. ...

### ディレクトリ構成案

App Routerではファイルコロケーションを強く意識した設計がなされているため、Route Segmentで利用するコンポーネントや関数はできるだけ近くに配置することが推奨されます。

Container/Presentationalパターンにおいて、ページやレイアウトから見える単位はContainer単位です。Presentational Componentsやその他のコンポーネントは、Container Componentsの実装詳細に過ぎません。

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
│  ├── _components
│  ├── _lib
│  └── ...
└── ...
```

命名やさらに細かい分割など詳細はプロジェクトごとに適宜修正してもいいと思いますが、App Routerらしい設計としてはディレクトリをContainer単位で、ファイルをサーバー・クライアントの境界観点から分割することが重要だと筆者は考えます。

## トレードオフ

### テストのためのexport

前述のように、Presentational ComponentsはContainer Componentsの実装詳細と捉えることもできるので、本来Presentational Componentsはプライベート定義として扱うことが好ましいと考えられます。

[ディレクトリ構成例](#ディレクトリ構成例)に基づいた設計の場合、Presentational Componentsは`presentational.tsx`で定義されます。

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

上記のようなディレクトリ設計に沿わない場合でも、Presentational ComponentsはContainer Componentsのみが利用しうる実質的なプライベート定義として扱うようにしましょう。
