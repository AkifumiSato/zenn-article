---
title: "UIをツリーに分解する"
---

## 要約

Reactの基本的な設計思想は、UI^[ここでのUIとは、データフェッチ等を含むReactコンポーネントで行う全てのことを含みます。]をコンポーネントのツリーで設計することです。ページやレイアウトなどの実装は、**UIをツリーに分解する**ことから始めましょう。これにより、データフェッチコロケーションやCompositionパターンの早期適用を目指します。

:::message
本章の内容は、React公式ドキュメントの[UIをツリーに分解する↗︎](https://ja.react.dev/learn/understanding-your-ui-as-a-tree#your-ui-as-a-tree)の理解が前提です。自信のない方はこちらを先にご参照ください。
:::

## 背景

[第1部 データフェッチ](part_1)でServer Componentsの設計パターンを、[第2部 コンポーネント設計](part_2)ではここまでClient Componentsの設計パターンを解説してきました。特に、[データフェッチ コロケーション](part_1_colocation)や[Compositionパターン](part_2_composition_pattern)は、後から適用しようとすると大きな手戻りを生む可能性があるため、早期から考慮して設計することが重要です。

## 設計・プラクティス

データフェッチコロケーションとCompositionパターンを早期適用するには、トップダウンな設計が効果的です。レイアウトやページといったUIを**ツリーに分解する**ことから始め、コンポーネントツリーの実装、各コンポーネントの詳細実装という流れで実装しましょう。

### 「大きなコンポーネント」と「小さなコンポーネント」

ReactではUIをコンポーネントとして表現します。ページやレイアウトなどのUIは「大きなコンポーネント」であり、「小さなコンポーネント」を組み合わせて^[ここでの「大きい」「小さい」とは、コンポーネントの位置関係を表すものです。コンポーネントの実装のサイズ（行数）ではありません。]実装します。RSCにおいてはServer Componentsという新たな種類のコンポーネントを組み合わせることができるようになりましたが、「小さなコンポーネント」を組み合わせて「大きなコンポーネント」を表現するという基本的な設計思想は変わりません。

「大きなコンポーネント」はボトムアップに実装すると手戻りが多くなりやすいため、筆者はトップダウンに設計することを推奨します。

### 実装手順

具体的には、以下のように進めることを推奨します。

1. 設計: **UIをツリーに分解する**
2. 仮実装: コンポーネントのツリーを仮実装
3. 実装: 各コンポーネントの詳細実装
   a. Server Componentsを実装
   b. Shared/Client Componentsを実装

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

これらのデータの取得には、以下のAPIを利用するものとします。

- PostAPI: 投稿IDをもとにブログ記事情報を取得するAPI
- UserAPI: ユーザーIDをもとにユーザー情報を取得するAPI
- CommentsAPI: 投稿IDをもとにコメント一覧を取得するAPI

#### 1. UIをツリー構造に分解する

UIを画面の要素が依存するデータを元にツリーに分解します。

![APIの依存関係](/images/nextjs-basic-principle/component-tree-example.png)

#### 2. コンポーネントのツリーを仮実装

上記の図をもとに、分解したUIの各要素をServer Componentsとして仮実装します。ここでデータフェッチするコンポーネントを`{Name}Container`という命名で仮実装します^[`Container`という命名は、[Container/Presentationalパターン](part_2_container_presentational_pattern)を元にしています。]。

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

#### 3. 各コンポーネントの詳細実装

以降は、仮実装となっているContainer Componentsの詳細な実装を行い、UIを完成させます。

各コンポーネントの詳細な実装は主題ではないため、本章では省略します。

::::details 各コンポーネントの実装イメージ

以下はContainer Componentsの実装イメージです。データフェッチ層などの実装は省略しています。

```tsx
// `/posts/[postId]/_containers/post/container.tsx`
export async function PostContainer({
  postId,
  children,
}: {
  postId: string;
  children: React.ReactNode;
}) {
  const post = await getPost(postId); // Request Memoization

  return (
    <div>
      {/* ...省略... */}
      {children}
      {/* ...省略... */}
    </div>
  );
}

// `/posts/[postId]/_containers/user-profile/container.tsx`
export async function UserProfileContainer({ postId }: { postId: string }) {
  const post = await getPost(postId); // Request Memoization
  const user = await getUser(post.authorId);

  return <div>{/* ...省略... */}</div>;
}

// `/posts/[postId]/_containers/comments/container.tsx`
export async function CommentsContainer({ postId }: { postId: string }) {
  const comments = await getComments(postId);

  return (
    <div>
      {comments.map((comment) => (
        <div key={comment.id}>{/* ...省略... */}</div>
      ))}
    </div>
  );
}

async function CommentItemContainer({ comment }: { comment: Comment }) {
  const user = await getUser(comment.authorId); // `getUser`は内部的にDataLoaderを利用

  return <div>{/* ...省略... */}</div>;
}
```

::::

## トレードオフ

### 重複するデータフェッチやN+1データフェッチ

UIを「小さなコンポーネント」に分解し、末端のコンポーネントでデータフェッチを行うことは重複リクエストのリスクが伴います。[Request Memoization](part_1_request_memoization)の章で解説したように、Next.jsではRequest Memoizationによってレンダリング中の同一リクエストを排除するため、データフェッチ層の設計が重要です。

また、配列を扱う際には[DataLoader](part_1_data_loader)を利用することで、N+1データフェッチを解消することができます。
