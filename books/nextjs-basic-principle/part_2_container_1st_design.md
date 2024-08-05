---
title: "Container 1stな設計"
---

## 要約

画面の設計はまずContainer Componentsのみでまず行い、Presentational Componentsは後から追加することを心がけましょう。

## 背景

[第1部](part_1)ではServer Componentsの設計パターンを、[第2部](part_2)ではここまでClient Componentsも含めたコンポーネント全体の設計パターンを解説してきました。ここまで順に読んでいただいた方はすでに多くの設計パターンを理解されてることと思います。

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

## トレードオフ

### テストのためのexport

上述のようなContainer 1stな設計では、Presentational ComponentsはContainer Componentsの実装詳細と捉えることもできます。そのため、Presentational ComponentsはContainer Componentsを定義するモジュールのプライベート定義として扱うことが好ましいとも考えられます。

```tsx :app/post.tsx
export async function PostContainer({ postId, children }: { postId: string }) {
  const post = await getPost(postId);

  return (
    <PostPresentation post={post}>
      <UserProfileContainer id={post.userId} />
    </PostPresentation>
  );
}

function PostPresentation({
  post,
  children,
}: {
  post: Post;
  children: ReactNode;
}) {
  // ...
}
```

上記の例では`<PostPresentation>`はexportされておらず、`post.tsx`のプライベート定義となっています。しかし、このままではStorybookやReact Testing Libraryで扱うことができないので、`<PostPresentation>`はexportする必要があります。

本来、筆者はテストのためにこういったコンポーネントや関数をexportすることはあまりいい手段だと思っていません。プロジェクト全体での認知負荷が上がり、補完のノイズにもなるからです。しかし、この設計のメリットであるテスト容易性を得るためには、現状Presentational Componentsをexportするトレードオフを受け入れる必要があります。

このトレードオフはあくまで**exportせざるを得ない**ことであって、publicなコンポーネントとして**複数のContainer Componentsで扱うことは避けるべき**です。Presentational Componentsは実装上はpublicでも実質的privateなものとして扱いましょう。
