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

データフェッチの設計には大きく２パターンあると筆者は考えています。データフェッチ層を設けるなどするような**中央集権型**の設計と、データフェッチコロケーション^[コードをできるだけ関連性のある場所に配置することを指します。]と称される**自立分散型**の設計です。

:::message
MetaやReactにおける自律分散型の設計の歴史については、筆者の前回の記事[Reactチームが見てる世界、Reactユーザーが見てる世界](https://zenn.dev/akfm/articles/react-team-vision)を参照ください。
:::

冒頭述べたようにMetaでは自律分散型の設計が重視されています。データフェッチ層を設けるような中央集権型の設計はなぜ好まれないのでしょう？

以降は、データフェッチ層やデータフェッチコロケーションの実装を見ながら問題点について考察します。

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

  // ...`posts`を参照
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

  // ...`richPosts`を参照
}
```

データフェッチは計4回、うち3つは`Promise.all()`によって並行化することでデータフェッチは2段階に整理されており、無駄のないデータフェッチ設計になっています。

一方で、保守性の観点で言うとどうでしょう？おそらく人によって様々だと思うのですが、筆者にとっては依存関係が複雑で読みづらいと感じます。さらにデータフェッチを数個増やしたいとなったら、どう修正するか考えるのも面倒だと感じる人は多いことでしょう。

:::details 中央集権的と逆行する関数分離
「複雑に感じるなら関数に分離すればいい」という考え方もあるかもしれませんが、中央集権的な設計は中央で読めることに価値があるため、はいい解決策にはならないと筆者は考えます。

例えば複数のデータフェッチを1つの関数に集約すると、データフェッチ層でどれだけデータフェッチを行なっているか分かりづらくなります。

```tsx
export async function Page(props: {
  searchParams: Promise<{ page?: string }>;
}) {
  const searchParams = await props.searchParams;
  const richPosts = await fetchAllRichPosts();

  // ...`richPosts`を参照
}
```

この場合、`fetchAllRichPosts()`でどれだけリクエストが走ったのか不透明です。`fetchAllRichPosts()`には含めたくないが、共有したいデータフェッチが増えるとしたら、修正は非常に面倒になるでしょう。

データフェッチ以外を関数抽出するアプローチも考えられます。

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
  const richPosts = mergePosts({
    posts,
    allAuthors,
    commentCountsMap,
    viewCountsMap,
  });

  // ...`richPosts`を参照
}
```

元の実装と比べ、極端に短くはなりません。また、UIが参照してるデータの由来を調べる際には、`mergePosts()`の実装を確認する必要があります。

このように、関数に抽出するだけではおおよそ本質的な保守性の改善は見込めません。また、これらのアプローチはシンプルなルールにしづらいため、一貫性の欠如にも繋がりやすいと筆者は考えます。
:::

## データフェッチコロケーションの弊害

一方、データフェッチコロケーションを採用すると、コードの見通しがとてもよくなります。

```tsx
// post-cad.tsx
export async function PostCard({ post }: { post: Post }) {
  const [authors, comments, viewCount] = await Promise.all([
    fetchAuthorsByIds(post.authorIds),
    fetchCommentCount(post.id),
    fetchViewCount(post.id),
  ]);

  // ...`authors`, `comments`, `viewCount`を参照
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

しかし一方で、パフォーマンス観点では非常に大きな問題が発生します。`fetchAuthors()`が`fetchAuthorsByIds()`などに変更されており、`post`分データフェッチされるように修正されています。これにより、修正前は4回だったデータフェッチが`posts`の取得1回+`posts`の取得分×3回分に増えており、典型的なN+1を引き起こしています。もし`post`が100件だった場合、301回分のデータフェッチが発生します。

データフェッチはパフォーマンス観点でボトルネックになりやすい部分です。データフェッチの極端な増加は、無視できない問題です。

## データフェッチのバッチング

ここまでの話を整理してみます。データフェッチ層を設けるよりデータフェッチコロケーションの方が、保守性には優れています。しかし、データフェッチコロケーションでは無視できないパフォーマンス問題を引き起こす可能性が高いと言えます。ある程度の保守性とパフォーマンスはトレードオフするしかないのでしょうか？

Metaではこの問題を**バッチング**によって解決しています。バッチングとは、複数のデータフェッチを1つにまとめて、効率的に処理する機構です。これを容易に実現するために、Metaは[DataLoader](https://www.npmjs.com/package/dataloader)をOSSで提供しています。

```ts
async function authorsBatch(authorIds: readonly string[]) {
  // keysを元にデータフェッチ
  const allAuthors = fetchAuthors(authorIds);
  return authorIds.map(
    (authorId) => allAuthors.find((author) => author.id === authorId) ?? null,
  );
}

// const authorLoader = new DataLoader(authorsBatch);
// authorLoader.load("1");
// authorLoader.load("2");
// 呼び出しはDataLoaderによってまとめられ、`authorsBatch(["1", "2"])`が呼び出される
```

これにより、前述のようなN+1問題を解決することができます。

```tsx
// 予期せぬキャッシュ共有をしないよう、`React.cache()`でリクエスト単位のインスタンス生成
const getAuthorLoader = React.cache(() => new DataLoader(authorsBatch));

async function fetchAuthorsByIds(authorId: string) {
  const authorLoader = getAuthorLoader();
  return authorLoader.load(authorId);
}
```

このように`fetchAuthorsByIds()`を実装すれば、データフェッチコロケーションしつつバッチングによりN+1が防げます。驚くべきことに、`fetchAuthorsByIds()`の使い方は何一つ変わりません。

```tsx
// post-cad.tsx
export async function PostCard({ post }: { post: Post }) {
  const [authors, comments, viewCount] = await Promise.all([
    fetchAuthorsByIds(post.authorIds),
    fetchCommentCount(post.id),
    fetchViewCount(post.id),
  ]);

  // ...`authors`, `comments`, `viewCount`を参照
}
```

:::message
React Routerの作者であるRyan Florence氏によって作られた、[batch-loader](https://github.com/ryanflorence/batch-loader)はDataLoaderに大きく影響を受けています。
:::

## 短期キャッシュ

データフェッチコロケーションでよく発生する問題がもう1つあります。同一リクエストです。

現在ログインしてるユーザー情報を取得する`fetchCurrentUser()`を用いて、ヘッダーにログイン時にアイコンを表示するとします。

```tsx
export async function UserIcon() {
  const user = await fetchCurrentUser();

  // ...`user`を参照
}
```

同様に、コメントを追加する際にはログイン状態を参照する必要があり、`fetchCurrentUser()`を実行する必要があるとします。

```tsx
export async function AddComment() {
  const user = await fetchCurrentUser();

  // ...`user`を参照
}
```

これらのコンポーネントは離れているため、`<Suspense>`境界によってレンダリングタイミングが異なる可能性があり、必ずしもバッチングできるとは限りません。DataLoaderにはキャッシュ機能が存在するので、バッチングできずともキャッシュによってこの問題は解決しえますが、データフェッチは必ずDataLoaderを介することとするのは冗長さを否めません。

このような問題を容易に解決するためにReactが提供しているのが、[React Cache](https://ja.react.dev/reference/react/cache)です。DataLoaderの実装例でもDataLoaderのインスタンス保持のために利用していました。

React Cacheを用いた`fetchCurrentUser()`の実装例は以下です。

```tsx
export const fetchCurrentUser = React.cache(async () => {
  const cookies = await cookies();
  const sessionId = cookieStore.get("session-id");
  const user = await fetch(`https://.../?sessionId=${sessionId}`);
  return user;
});
```

ただし、実際にはNext.jsの[Request Memoization](https://nextjs.org/docs/app/deep-dive/caching#request-memoization)のように、フレームワーク側で同一リクエストの排除が実装されていることが多いと考えられます。そのため、`fetch()`の場合には`React.cache()`すら不要になるでしょう。

## まとめ

Metaでは、自律分散的な設計にバッチングと短期キャッシュを組み合わせることによって、パフォーマンスと保守性を両立させています。これは大規模開発のみで有用なアプローチではなく、小規模な開発から適用可能で保守性を向上させる有効な手段です。

実際に、筆者は昨今好んでDataLoaderやReact Cacheを利用して自律分散的な設計を心がけていますが、メリットを感じる場面が多く、筆者にとってこれらは必要不可欠な存在です。

また、より詳細にNext.jsにおけるDataLoaderの使い方を知りたい場合には、筆者が以前執筆した「Next.jsの考え方」の以下の章をご参照ください。

https://zenn.dev/akfm/books/nextjs-basic-principle/viewer/part_1_data_loader
