---
title: "Metaに学ぶ、大規模開発のデータフェッチ設計と最適化"
emoji: "🧠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "設計"]
published: true
---

データフェッチの設計は、保守性とパフォーマンスに強く影響します。これらはよくトレードオフに扱われ、パフォーマンスを優先すると保守性が犠牲に、保守性を優先するとパフォーマンスが犠牲になりがちです。

Metaでは大規模開発における保守性とパフォーマンスの両立を目指し、研究が行われてきました。本稿では、Metaが重視している自律分散型のデータフェッチ設計と、それを支えるバッチングと短命のキャッシュについて解説します。

:::message

- 本稿はReactチームのSophie Alpert氏のブログ記事、[Fast and maintainable patterns for fetching from a database](https://sophiebits.com/2020/01/01/fast-maintainable-db-patterns)を参考にしています。
- 上記ブログはタイトルに「データベース」とありますが、考え方はデータベースアクセスに限りません。本稿ではReactを実装例に用いて解説します。

:::

## 要約

- 大規模開発における保守性には、自律分散型の設計思想が重要
- 無策にデータフェッチコロケーションすると、パフォーマンスがトレードオフになる
- Metaではバッチングと短命のキャッシュによって、保守性とパフォーマンスがトレードオフにならないようにしてる
- [DataLoader](https://www.npmjs.com/package/dataloader)や[React Cache](https://ja.react.dev/reference/react/cache)は、これらを容易に実現する手段

## データフェッチの設計パターン

筆者の考えでは、データフェッチの設計は大きく2パターンに分けられます。データフェッチ層を設けるなどするような**中央集権型**の設計と、データフェッチコロケーション^[コードをできるだけ関連性のある場所に配置することを指します。]に代表される**自律分散型**の設計です。

:::message
MetaやReactにおける自律分散型の設計の歴史については、[Reactチームが見てる世界、Reactユーザーが見てる世界](https://zenn.dev/akfm/articles/react-team-vision)で詳細に解説しています。興味のある方はご参照ください。
:::

冒頭で触れたように、Metaでは自律分散型の設計が重視されており、特に大規模開発の保守性において重要だと考えられています。データフェッチ層を設けるような中央集権型の設計はなぜ好まれないのでしょう？

実装例を元に問題点を考察してみましょう。

## 中央集権型の弊害

以下はNext.jsでブログ記事一覧ページを実装する際、Pageコンポーネントを中央集権的なデータフェッチ層として扱う実装例です。

```tsx
export async function Page(props: {
  searchParams: Promise<{ page?: string }>;
}) {
  const searchParams = await props.searchParams;
  const page = searchParams.page ? Number(searchParams.page) : 1;
  const posts = await fetchPosts({
    page,
  }); // Post[]

  // ...`posts`を参照
}

type Post = {
  id: string;
  title: string;
  authorIds: string[];
  summary: string;
};
```

これは非常にシンプルでわかりやすい例です。このままでも特に問題ないでしょう。

しかし、このままでは`Post`に含まれている情報が少なく、ブログ一覧として出せる情報も少なすぎるので、以下の情報を追加で表示する改修をするとします。

- 著者情報
- コメント数
- 閲覧数

なお、これらは`fetchPosts()`の結果には含まれず、**それぞれ別なAPIからデータを取得するもの**とします。実装例においては少々やりすぎに見えますが、APIで責務過多な状態はGod API（神API）と呼ばれるアンチパターンのため、これを避けるために細粒度でRestful準拠を重視していることを想定します。

以下は改修後の実装例です。

```tsx
export async function Page(props: {
  searchParams: Promise<{ page?: string }>;
}) {
  const searchParams = await props.searchParams;

  // ベースとなるブログ一覧を取得
  const page = searchParams.page ? Number(searchParams.page) : 1;
  const posts = await fetchPosts({
    page,
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

  // `richPosts: RichPost[]`を組み立て
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

データフェッチは計4回、うち3つは`Promise.all()`によって並行化することでデータフェッチは2段階に整理されており、God APIを避けつつある程度最適化されたデータフェッチ設計になっています。

一方、保守性の観点で言うとどうでしょうか？おそらく人によって様々だと思うのですが、筆者は依存関係が複雑で読みづらいと感じます。この程度なら許容範囲内という人でも、組み合わせる配列の数が更に増えていくと保守性に乏しいと感じるのではないでしょうか。

:::details 中央集権的と逆行する関数分離
「複雑に感じるなら関数に分離すればいい」という考え方もあるかもしれませんが、中央集権的な設計は**集権された層で管理できることに価値がある**ため、関数分離は良い解決策にはならないと筆者は考えます。

例えば複数のデータフェッチを1つの関数に集約すると、データフェッチ層でどれだけデータフェッチを行っているか分かりづらくなります。

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
  const page = searchParams.page ? Number(searchParams.page) : 1;
  const posts = await fetchPosts({
    page,
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

元の実装と比べ、極端に短くはなりません。また、UIが参照しているデータの由来を調べる際には、`mergePosts()`の実装を確認する必要があります。

このように、関数に抽出するだけではおおよそ本質的な保守性の改善は見込めません。また、これらのアプローチはシンプルなルールにしづらいため、一貫性の欠如にも繋がりやすいと筆者は考えます。
:::

## 自律分散型の弊害

一方、自律分散型の設計を採用し、必要なデータを必要な時に取得するデータフェッチコロケーションを適用すると、コードの見通しがとても良くなります。

具体的には、`<Page>`では必要なデータを全て揃えるようにするのではなく、記事一覧をループするのに最低限必要となる記事情報の取得のみを行います。著者情報・コメント数・閲覧数といった付加情報の取得は、実際にこれらが必要になる記事単位のコンポーネント`<PostCassette>`などで行います。

```tsx
// page.tsx
export async function Page(props: {
  searchParams: Promise<{ page?: string }>;
}) {
  const searchParams = await props.searchParams;
  const page = searchParams.page ? Number(searchParams.page) : 1;
  const posts = await fetchPosts({
    page,
  });

  // `posts`をループして`<PostCassette>`を組み立てる
}

// post-cassette.tsx
export async function PostCassette({ post }: { post: Post }) {
  const [authors, comments, viewCount] = await Promise.all([
    fetchAuthorsByIds(post.authorIds),
    fetchCommentCount(post.id),
    fetchViewCount(post.id),
  ]);

  // ...`authors`, `comments`, `viewCount`を参照
}
```

修正前と比べて非常に読みやすく、シンプルになりました。ファイル分割され、参照するデータも明確なため可読性が高く、修正時のデグレリスクも低いと考えられます。筆者の感覚では、保守性はこちらの方が圧倒的に良いと感じます。

しかし一方で、パフォーマンス観点では非常に大きな問題が発生します。`fetchAuthors()`が`fetchAuthorsByIds()`などに変更されており、著者情報の取得が`post`の数分データフェッチされるような設計になっています。これにより、修正前は4回だったデータフェッチが`posts`の取得1回+`posts`の取得分×3回分に増えており、典型的なN+1を引き起こしています。もし`post`が100件だった場合、301回分のデータフェッチが発生します。`fetchAuthorsByIds(post.authorIds)`には同一リクエストも含まれることでしょう。

データフェッチはパフォーマンス観点でボトルネックになりやすい部分です。このようなデータフェッチの極端な増加は、無視できない非常に大きな問題です。

## データフェッチのバッチング

ここまでの話を整理してみます。中央集権型の設計より自律分散型の設計の方が、保守性には優れています。しかし、自律分散型の設計では無視できないパフォーマンス問題を引き起こす可能性が高いと言えます。この場合、保守性とパフォーマンスはトレードオフするしかないのでしょうか？

Metaではこの問題を**バッチング**によって解決しています。バッチングとは、複数のデータフェッチを1つにまとめて、効率的に処理する機構です。具体的な仕組み^[[DataLoaderの実装にあるコメント](https://github.com/graphql/dataloader/blob/a10773043d41a56bde4219c155fcf5633e6c9bcb/src/index.js#L214-L239)が参考になります。]としては、Node.jsの`process.nextTick()`やブラウザ側の`setImmediate()`などを利用して、データフェッチするタイミングを「少し待つ」ことで、データフェッチのバッチングが実現できます。

これを容易に実現するため、Metaは[DataLoader](https://www.npmjs.com/package/dataloader)というライブラリをOSSで提供しています。

```ts
async function authorsBatch(authorIds: readonly string[]) {
  // `authorIds`を元にデータフェッチ
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
// post-cassette.tsx
export async function PostCassette({ post }: { post: Post }) {
  const [authors, comments, viewCount] = await Promise.all([
    fetchAuthorsByIds(post.authorIds),
    fetchCommentCount(post.id),
    fetchViewCount(post.id),
  ]);

  // ...`authors`, `comments`, `viewCount`を参照
}
```

`fetchCommentCount()`や`fetchViewCount()`もDataLoaderによってバッチングすれば、上記のような実装のままN+1を解決し、保守性とパフォーマンスを同時に得ることができます。

:::message
React Routerの作者であるRyan Florence氏によって作られた[batch-loader](https://github.com/ryanflorence/batch-loader)は、DataLoaderに大きく影響を受けています。機能をより狭めたもののようです。
:::

## 短命のキャッシュ

データフェッチコロケーションでよく発生する問題がもう1つあります。同一リクエストの重複実行です。

現在ログインしているユーザー情報を取得する`fetchCurrentUser()`を用いて、ヘッダーにログイン時にアイコンを表示するとします。

```tsx
export async function UserIcon() {
  const user = await fetchCurrentUser();

  // ...`user`を参照
}
```

同様に、コメント欄でコメントを追加するにはログイン状態を参照する必要があり、`fetchCurrentUser()`を実行する必要があるとします。

```tsx
export async function AddComment() {
  const user = await fetchCurrentUser();

  // ...`user`を参照
}
```

これらのコンポーネントは離れているため、`<Suspense>`境界によってレンダリングタイミングが異なる可能性があり、バッチングできるとは限りません。DataLoaderにはキャッシュ機能が存在するので、バッチングできずともキャッシュによってこの問題は解決し得ますが、バッチングの必要がないのにDataLoaderを介するのは冗長かつ不自然な実装になり得ます。

このような問題を容易に解決するためにReactが提供しているのが、[React Cache](https://ja.react.dev/reference/react/cache)です。React Cacheはサーバーへのリクエストごとに作成される、**短命のキャッシュ**です。すでにDataLoaderの実装例でも、DataLoaderのインスタンス保持のために利用していました。React Cacheはリクエストごとに破棄されることによって、キャッシュサーバーやメモリリーク、予期せぬキャッシュ共有などを防ぐことができます。

React Cacheを用いた`fetchCurrentUser()`の実装例は以下です。

```tsx
export const fetchCurrentUser = React.cache(async () => {
  const cookieStore = await cookies();
  const sessionId = cookieStore.get("session-id");
  const { user } = await fetch(`https://.../?sessionId=${sessionId}`);
  return user;
});
```

ただし、実際にはNext.jsの[Request Memoization](https://nextjs.org/docs/app/deep-dive/caching#request-memoization)のように、フレームワーク側で同一リクエストの排除が実装されていることが多いと考えられます^[元々はReact側の機能として実装されていたはずですが、現状RSCの必須要件なのかは不明です。]。そのため、上記のように`fetch()`するのみなら`React.cache()`は不要になるでしょう。

## まとめ

Metaでは、自律分散的な設計にバッチングと短命のキャッシュを組み合わせることによって、パフォーマンスと保守性を両立させています。これは大規模開発のみで有用なアプローチではなく、小規模な開発から適用可能で有効な手段です。

実際に、筆者は小規模なアプリケーションでも好んでDataLoaderやReact Cacheを利用した自律分散的な設計を採用していますが、メリットを感じる場面が多く、筆者にとってこれらは必要不可欠な存在です。

本稿を通じて、バッチングや短命のキャッシュの有効性が伝われば幸いです。

### 余談: Next.jsにおけるDataLoaderとReact Cache

より詳細にNext.jsにおけるDataLoaderの使い方を知りたい方は、筆者が以前執筆した「Next.jsの考え方」の以下の章をご参照ください。

https://zenn.dev/akfm/books/nextjs-basic-principle/viewer/part_1_data_loader

https://zenn.dev/akfm/books/nextjs-basic-principle/viewer/part_1_request_memoization
