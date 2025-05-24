---
title: "Metaに学ぶ、大規模開発のデータフェッチ設計と最適化"
emoji: "🧠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "設計"]
published: true
---

Reactにおけるデータフェッチの設計は、保守性とパフォーマンスに強く影響します。これらはよくトレードオフに扱われ、パフォーマンスを優先すると保守性が犠牲に、保守性を優先するとパフォーマンスが犠牲になりがちです。

Metaでは大規模開発における保守性とパフォーマンスの両立を目指し、研究が行われてきました。本稿では、Metaが重視してるデータフェッチの自律分散型の設計と、それを支えるバッチングと短期キャッシュについて解説します。

:::message
本稿は、ReactチームのSophie Alpert氏のブログ[Fast and maintainable patterns for fetching from a database](https://sophiebits.com/2020/01/01/fast-maintainable-db-patterns)を参考にしています。より一次情報を参照したい方は、こちらをお読みください。
:::

## 要約

- 大規模開発における保守性には、自立分散的思考が重要
- 無策にデータフェッチコロケーションすると、パフォーマンスがトレードオフになる
- Metaではバッチングと短期キャッシュによって、データフェッチコロケーションしつつパフォーマンスをトレードオフにならないようにしてる
- [DataLoader](https://www.npmjs.com/package/dataloader)や[React Cache](https://ja.react.dev/reference/react/cache)は、これらを容易に実現する手段

## データフェッチの設計パターン

データフェッチの設計には大きく２パターンあると筆者は考えています。データフェッチ層を設けるなどするような**中央集権型**の設計と、データフェッチコロケーションと称される**自立分散型**の設計です。

:::message
MetaやReactにおける自律分散型の設計の歴史については、筆者の前回の記事[Reactチームが見てる世界、Reactユーザーが見てる世界](https://zenn.dev/akfm/articles/react-team-vision)を参照ください。
:::

冒頭述べたようにMetaでは自律分散型の設計が重視されています。データフェッチ層を設けるような設計はなぜ好まれないのでしょう？

## データフェッチ層の弊害

以下はNext.jsのページコンポーネントをデータフェッチ層として扱う場合の、ブログアプリケーションの実装例です。

```tsx
export async function Page(props: {
  searchParams: Promise<{ page?: string }>;
}) {
  const searchParams = await props.searchParams;
  const posts = await fetchPosts({
    page: searchParams.page ? Number(searchParams.page) : 1,
  });

  // ...`posts`を参照してUI組み立て
}

type Post = {
  id: string;
  title: string;
  authorIds: string[];
  summary: string;
};
```

これは非常にシンプルでわかりやすい例です。このままでも何も問題はないでしょう。しかしこのままでは`Post`に含まれてる情報が少なく、ブログ一覧として出せる情報も少なすぎるので、以下の情報を追加で表示する改修をするとします。

- 著者情報
- コメント数
- 閲覧数

なお、これらは`fetchPosts()`の結果には含まれず、それぞれ別なAPIからデータを取得するものとします。少々やりすぎに見えますが、APIで汎用性に乏しい状態はGod API（神API）とよばれるアンチパターンのため、これを避けるために細粒度なRestful準拠されてるものとします。

以下は変更後の実装例です。

```tsx
export async function Page(props: {
  searchParams: Promise<{ page?: string }>;
}) {
  const searchParams = await props.searchParams;
  // ベースとなるブログ一覧を取得
  const posts = await fetchPosts({
    page: searchParams.page ? Number(searchParams.page) : 1,
  });

  // ブログ一覧を補強する情報を一括で取得
  const postIds = posts.map((post) => post.id);
  const uniqueAuthorIds = Array.from(
    new Set(posts.flatMap((post) => post.authorIds)),
  );
  const [allAuthors, commentCountsMap, viewCountsMap] = await Promise.all([
    fetchAuthors(uniqueAuthorIds),
    fetchCommentCountsForPosts(postIds),
    fetchViewCountsForPosts(postIds),
  ]);

  // `RichPost`を組み立て
  const authorsMap = new Map(allAuthors.map((author) => [author.id, author]));
  const richPosts = posts.map((post) => {
    const authors = post.authorIds
      .map((id) => authorsMap.get(id))
      .filter((author) => author !== undefined);
    const comments = commentCountsMap.get(post.id) ?? 0;
    const viewCount = viewCountsMap.get(post.id) ?? 0;

    return {
      ...post,
      authors,
      comments,
      viewCount,
    };
  });

  // ...`richPosts`を参照してUI組み立て
}
```

データフェッチは計4回、うち3つは`Promise.all()`によって並行化することでデータフェッチは2段階に整理されており、無駄のないデータフェッチ設計になっています。

一方で、保守性の観点で言うとどうでしょう？おそらく人によって様々だと思うのですが、筆者にとっては依存関係が複雑で読みづらいと感じます。さらにデータフェッチを数個増やしたいとなったら、どう修正するか考えるのも面倒だと感じる人は多いことでしょう。

## データフェッチコロケーションの弊害

一方、データフェッチコロケーションを採用すると、コードの見通しがとてもよくなります。

```tsx
// post-cad.tsx
export async function PostCard({ post }: { post: PostBase }) {
  const [authors, comments, viewCount] = await Promise.all([
    fetchAuthorsByIds(post.authorIds),
    fetchCommentCount(post.id),
    fetchViewCount(post.id),
  ]);

  // ...`authors`, `comments`, `viewCount`を参照してUI組み立て
}

// page.tsx
export async function Page(props: {
  searchParams: Promise<{ page?: string }>;
}) {
  const searchParams = await props.searchParams;
  const posts = await fetchPosts({
    page: searchParams.page ? Number(searchParams.page) : 1,
  });

  // `posts`をループして`<PostCard>`を組み立てる
}
```

修正前と比べて非常に読みやすく、シンプルになりました。保守性で言えばこちらの方が圧倒的に良いと筆者は感じます。

しかし一方で、パフォーマンス観点では最悪です。修正前は4回だったデータフェッチが、こちらの実装例では`posts`の取得1回+`posts`の取得分×3回分発生しており、典型的なN+1を引き起こしています。

## データフェッチのバッチング

- ここまでの問題を整理すると、保守性観点での理想はデータフェッチコロケーションだが、パフォーマンスがトレードオフになってることである
- 実装例で言うと1つ目の問題は、典型的なN+1問題である
- Metaではバッチングを使って、「少し待つ」ことでデータフェッチコロケーションのままパフォーマンスを最適化してる
- これを容易に実現できるライブラリが、DataLoaderである
- （実装例）
- これにより、N+1が解消された
- 余談: RRの作者も最近似たようなのを出した

## 短期キャッシュ

- もう一つ問題がある。同一リクエストだ。
- コロケーションによって全く同一なリクエストが発行されるリスクが出てきた
- （実装例）
- これはバッチングでは解消されない
- これを解決するのはインメモリで短期的なキャッシュだ
- React Cacheはまさにこれを体現したものである
- （実装例）
- これにより、コロケーションで発生する大きな2つの問題を解決できる

## まとめ

- Metaでは、短期キャッシュとバッチングによってパフォーマンスと保守性を両立させてる
- これは大規模開発のみならず、小規模な開発から適用可能な優れた設計
- 実際、僕も普段から好んで採用してるが、複雑なループが避けられるのでとても素晴らしいと感じてる
- (Next.jsの考え方へ誘導)

---

```tsx
export default async function Page() {
  const user = await fetchUserData();
  const userBlogs = await fetchUserBlogs(user.id);

  const enrichedBlogs = await Promise.all(
    userBlogs.map(async (blog) => {
      const authorsPromises = blog.authorIds.map((authorId) =>
        fetchAuthorInfo(authorId),
      );
      const authors = await Promise.all(authorsPromises);
      const [commentStats, impressionStats] = await Promise.all([
        fetchBlogCommentCount(blog.blogId),
        fetchBlogImpressionCount(blog.blogId),
      ]);
      return {
        ...blog,
        authors,
        commentCount: commentStats.commentCount,
        impressionCount: impressionStats.impressionCount,
      };
    }),
  );

  // ...
}
```
