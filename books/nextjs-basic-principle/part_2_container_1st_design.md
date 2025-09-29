---
title: "合成から考えるページコンポーネント"
---

## 要約

ページコンポーネントの実装は、細粒度なServer Componentsを合成しツリー構造を設計することから始めましょう。これにより、データフェッチコロケーションやCompositionパターンの早期適用を目指します。

## 背景

[第1部 データフェッチ](part_1)でServer Componentsの設計パターンを、[第2部 コンポーネント設計](part_2)ではここまでClient Componentsの設計パターンを解説してきました。特に、[データフェッチ コロケーション](part_1_colocation)や[Compositionパターン](part_2_composition_pattern)は、後から適用しようとすると大きな手戻りを生む可能性があるため、早期から考慮して設計することが重要です。

## 設計・プラクティス

データフェッチコロケーションとCompositionパターンを早期適用するには、ページコンポーネントのツリー構造を設計することから始めることが効果的です。ツリー構造の設計は、**細粒度なServer Componentsを合成**することを意識しましょう。

### 設計手順

具体的な手順は以下の通りです。

1. コンポーネントの合成を考え、ツリー構造を仮実装
2. Server Componentsを実装
3. Shared/Client Componentsを実装

:::message
最初に決めた設計に固執する必要はありません。実装を進める中でコンポーネントツリーを見直すことも重要です。
:::

### 設計例

例として、ブログ記事画面について「1. コンポーネントの合成を考え、ツリー構造を仮実装」のステップを実施してみます。ブログ記事画面は以下の要素で構成されているものとします。

- ブログ記事情報
- 著者情報
- コメント一覧

これらのデータソースとして、以下のAPIを利用するものとします。

- PostAPI: 投稿IDをもとにブログ記事情報を取得するAPI
- UserAPI: ユーザーIDをもとにユーザー情報を取得するAPI
- CommentsAPI: 投稿IDをもとにコメント一覧を取得するAPI

以下はこれらの画面要素とAPIの依存関係を図示したものです。

![APIの依存関係](/images/nextjs-basic-principle/component-tree-example.png)

上記の図をもとに、画面の構成要素をServer Componentsとして実装イメージを書き出します。

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

:::message
`Container`という命名は**Container/Presentationalパターン**におけるContainerコンポーネントを意図したものです。詳細は後述の[Container/Presentationalパターン](part_2_container_presentational_pattern)にて解説します。
:::

`<PostContainer>`や`<CommentsContainer>`などのコンポーネントは仮実装で構いません。これで「1. コンポーネントの合成を考え、ツリー構造を仮実装」は完了です。あとはステップ2,3を繰り返し、各コンポーネントの詳細を実装しましょう。実装しながら適宜Server Componentsのツリー構造を見直して、必要に応じて修正することもできます。

::::details 各コンポーネントの実装イメージ

各Server Components（Container Components）の実装イメージは以下の通りです。データフェッチ層やPresentational Componentsの実装は省略しています。

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

### 細粒度なServer Componentsとファイルコロケーション

Next.jsの規約ファイルはコロケーションを強く意識した設計がなされており、[Route Segment↗︎](https://nextjs.org/docs/app/getting-started/layouts-and-pages#creating-a-nested-route)で利用するコンポーネントや関数もできるだけコロケーションすることが推奨^[参考: [公式ドキュメント↗︎](https://nextjs.org/docs/app/getting-started/project-structure#colocation)]されます。

前述の例では[Private Folder↗︎](https://nextjs.org/docs/app/getting-started/project-structure#private-folders)の機能を利用して、Container単位で`_containers`ディレクトリにコロケーションすることができます。

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

前述のディレクトリ構成例における`presentational.tsx`など、コロケーションしたファイルには実質的にPrivateなファイルが含まれます。しかし、JavaScriptでは`export`にスコープを設定することはできません。

`presentational.tsx`のように実質的にPrivateなファイルは、[eslint-plugin-import-access↗︎](https://github.com/uhyo/eslint-plugin-import-access)やbiomeの[`noPrivateImports`↗︎](https://biomejs.dev/linter/rules/no-private-imports/)を利用することで、`import`可能なスコープを制限することができます。
