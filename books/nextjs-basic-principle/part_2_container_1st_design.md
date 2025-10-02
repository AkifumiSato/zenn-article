---
title: "UIをツリーに分解する"
---

## 要約

ページやレイアウトなどの実装は、**UIをツリーに分解する**ことから始めましょう。これにより、データフェッチコロケーションやCompositionパターンの早期適用を目指します。

:::message
本章の内容は、React公式ドキュメントの[「UIをツリーに分解する」](https://ja.react.dev/learn/understanding-your-ui-as-a-tree#your-ui-as-a-tree)の理解が前提です。自信のない方はこちらを先にご参照ください。
:::

## 背景

[第1部 データフェッチ](part_1)でServer Componentsの設計パターンを、[第2部 コンポーネント設計](part_2)ではここまでClient Componentsの設計パターンを解説してきました。特に、[データフェッチ コロケーション](part_1_colocation)や[Compositionパターン](part_2_composition_pattern)は、後から適用しようとすると大きな手戻りを生む可能性があるため、早期から考慮して設計することが重要です。

## 設計・プラクティス

データフェッチコロケーションとCompositionパターンを早期適用するには、**UIをツリーに分解する**ことから始めるのが効果的です。ツリー構造は、データや振る舞いごとに細粒度に分割することを意識しましょう。

### 実装手順

具体的には、以下のように進めることを推奨します。

1. 設計: **UIをツリーに分解する**
2. 仮実装: React要素（主にServer Components）のツリーを仮実装
3. 実装: Server Componentsを実装
4. 実装: Shared/Client Componentsを実装

:::message
最初に決めたツリー構造に固執する必要はありません。実装を進める中でツリーを見直すことも重要です。
:::

### 実装例

以下のようなブログ記事画面を例として考えてみます。

![UIをツリー構造に分解する](/images/nextjs-basic-principle/blog-ui-example.png =400x)

この画面は以下のような要素で構成されています。

- ブログ記事情報
- 著者情報
- コメント一覧

これらに対応するデータソースとして、以下のAPIを利用するものとします。

- PostAPI: 投稿IDをもとにブログ記事情報を取得するAPI
- UserAPI: ユーザーIDをもとにユーザー情報を取得するAPI
- CommentsAPI: 投稿IDをもとにコメント一覧を取得するAPI

#### 1. UIをツリー構造に分解する

画面の要素をデータソースの依存関係を元に、UIをツリーに分解します。

![APIの依存関係](/images/nextjs-basic-principle/component-tree-example.png)

#### 2. React要素のツリーを仮実装

上記の図をもとに、分解したUIの各要素をServer Componentsとして仮実装します。ここでは[Container/Presentationalパターン](part_2_container_presentational_pattern)を元に、各Server Componentsを`{Name}Container`という命名で仮実装します。

```tsx:/posts/[postId]/page.tsx
export default async function Page(props: {
  params: Promise<{ postId: string }>;
}) {
  const { postId } = await props.params;

  return (
    <div className="flex flex-col gap-4">
      <PostContainer postId={postId}>
        <UserProfileContainer postId={postId} />
      </PostContainer>
      <CommentsContainer postId={postId} />
    </div>
  );
}
```

#### 3,4: 各コンポーネントの詳細実装

以降は、仮実装となっているContainer Componentsの詳細な実装を行い、UIを完成させます。

各コンポーネントの詳細な実装は主題ではないため、本章では省略します。

::::details 各コンポーネントの実装イメージ

以下はContainer Componentsの実装イメージです。データフェッチ層やPresentational Componentsは省略しています。

```tsx
// `/posts/[postId]/_containers/post/container.tsx`
export async function PostContainer({ postId }: { postId: string }) {
  const post = await getPost(postId); // Request Memoization

  return <PostPresentation post={post} />;
}

// `/posts/[postId]/_containers/user-profile/container.tsx`
export async function UserProfileContainer({ postId }: { postId: string }) {
  const post = await getPost(postId); // Request Memoization
  const user = await getUser(post.authorId);

  return <UserProfilePresentation user={user} />;
}

// `/posts/[postId]/_containers/comments/container.tsx`
export async function CommentsContainer({ postId }: { postId: string }) {
  const comments = await getComments(postId);

  return (
    <CommentLayout>
      {comments.map((comment) => (
        <CommentItemContainer key={comment.id} comment={comment} />
      ))}
    </CommentLayout>
  );
}

async function CommentItemContainer({ comment }: { comment: Comment }) {
  const user = await getUser(comment.authorId); // `getUser`は内部的にDataLoaderを利用

  return <CommentItemPresentation comment={comment} user={user} />;
}
```

:::message

- `<PostContainer>`や`<CommentsContainer>`は`getPost(postId)`をそれぞれ呼び出しますが、[Request Memoization](part_1_request_memoization)によって重複リクエストは発生しません。
- `<CommentsContainer>`のように、ContainerとPresentationalが必ずしも1対1になるとは限りません。

:::

::::

### Container単位のディレクトリ構成例

Next.jsはファイルコロケーションを強く意識して設計されており、[Route Segment↗︎](https://nextjs.org/docs/app/getting-started/layouts-and-pages#creating-a-nested-route)で利用するコンポーネントや関数もできるだけコロケーションすることが推奨^[参考: [公式ドキュメント↗︎](https://nextjs.org/docs/app/getting-started/project-structure#colocation)]されます。上記手順で得られたページやレイアウトを構成するContainer Componentsも、同様にコロケーションすることが望ましいと考えられます。

以下は、筆者が推奨するディレクトリ構成の例です。[Private Folder↗︎](https://nextjs.org/docs/app/getting-started/project-structure#private-folders)を利用して、Container単位で`_containers`ディレクトリにコロケーションします。

```
/posts/[postId]
├── page.tsx
├── layout.tsx
└── _containers
    ├── post
    │  ├── index.tsx // Container Componentsをexport
    │  ├── container.tsx
    │  ├── presentational.tsx
    │  └── ... // その他のコンポーネントやUtilityなど
    ├── user-profile
    │  ├── index.tsx // Container Componentsをexport
    │  ├── container.tsx
    │  ├── presentational.tsx
    │  └── ... // その他のコンポーネントやUtilityなど
    └── comments
       ├── index.tsx // Container Componentsをexport
       ├── container.tsx
       ├── presentational.tsx
       └── ... // その他のコンポーネントやUtilityなど
```

コロケーションしたファイルは、外部から参照されることを想定した実質的にPublicなファイルと、Privateなファイルに分けることができます。上記の例では、`index.tsx`でContainer Componentsを`export`することを想定しています。

## トレードオフ

### 広すぎるexport

前述のように、Presentational ComponentsはContainer Componentsの実装詳細と捉えることもできるので、本来プライベート定義として扱うことが好ましいと考えられます。[Container単位のディレクトリ構成例](#container単位のディレクトリ構成例)では、Presentational Componentsは`presentational.tsx`で定義されます。

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

このように、同一ディレクトリにおいてのみ利用することを想定したモジュール分割においては、[eslint-plugin-import-access↗︎](https://github.com/uhyo/eslint-plugin-import-access)やbiomeの[`noPrivateImports`↗︎](https://biomejs.dev/linter/rules/no-private-imports/)を利用すると予期せぬ外部からの`import`を制限することができます。

上記のようなディレクトリ設計に沿わない場合でも、Presentational ComponentsはContainer Componentsのみが利用しうる**実質的なプライベート定義**として扱うようにしましょう。
