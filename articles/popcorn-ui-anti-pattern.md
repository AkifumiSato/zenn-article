---
title: "ポップコーンUIとReact"
emoji: "🍿"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "ui"]
published: false
---

ページアクセス時に複数のローディングスピナーがランダムに表示され、徐々にコンテンツに置き換わっていくような体験に遭遇したこと、もしくは実装した経験はあるでしょうか？ReactチームはこのようなUIを、ポップコーンが弾ける様子に例えて**ポップコーンUI**と揶揄しています。

このようなUIはユーザー体験として好ましくありませんが、よくみられるUIでもあります。Reactにおいて、コンポーネント内でデータフェッチを扱う方法は様々ありますが、複数のコンポーネントでローディング状態をハンドリングしてしまうとポップコーンUIになりがちです。

開発者が意図してより良い体験を実装すべきとも考えられますが、単独のコンポーネントで見ると自然な実装になってしまうことは構造的な問題とも捉えられます。特に、AI Agent時代においてはコンポーネント単体を考慮する機会が多いと考えられるので、AI Agentがシンプルにコンポーネントのみを考慮して実装した結果、ポップコーンUIになることもあるでしょう。

この記事ではポップコーンUIの実装について整理しつつ、ポップコーンUIを避けるための考え方やTipsを紹介します。

:::message
この記事ではデータフェッチライブラリとして[Tanstack Query](https://tanstack.com/query/latest)、メタフレームワークとして[Next.js](https://nextjs.org/)を例に解説しています。
:::

## ポップコーンUI

ポップコーンUIは冒頭で述べたように、UIがポップコーンのようにランダムに置き換わっていくような体験のことを指します。これはパフォーマンス観点では、Core Web Vitalsの1つである[Cumulative Layout Shift (CLS)](https://web.dev/articles/cls?hl=ja)という指標で定量化して計測可能です。

![Popcorn UI](/images/popcorn-ui-anti-pattern/use-query-with-popcorn-ui.gif)

### 実装例

以下は[Tanstack Query](https://tanstack.com/query/latest)を使ったポップコーンUIの実装例です。

```tsx
// 親はただ並べるだけ → 3つのスピナーがバラバラに解決される
export default function DashboardPage() {
  return (
    <main>
      <h1>Dashboard</h1>
      <UserProfile />
      <UserPosts />
      <UserStats />
    </main>
  );
}

// 各コンポーネントが個別にfetch → ポップコーンUI
function UserProfile() {
  const { data, isLoading } = useQuery({
    queryKey: ["user"],
    queryFn: fetchUser,
  });
  if (isLoading) return <Spinner />;
  return <div>{data.name}</div>;
}

function UserPosts() {
  const { data, isLoading } = useQuery({
    queryKey: ["posts"],
    queryFn: fetchPosts,
  });
  if (isLoading) return <Spinner />;
  return (
    <ul>
      {data.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}

function UserStats() {
  const { data, isLoading } = useQuery({
    queryKey: ["stats"],
    queryFn: fetchStats,
  });
  if (isLoading) return <Spinner />;
  return <div>{data.totalPosts} posts</div>;
}
```

このように、データを参照するコンポーネント内でデータフェッチを扱いつつ、ローディング状態を管理するとポップコーンUIになってしまいます。

### Propsのバケツリレー問題

前述のTanstack Queryを使った実装例は、末端のコンポーネントでデータフェッチとローディングを扱っているが故にポップコーンUIになります。親コンポーネントでこれらを中央集権的に管理すれば、統合的なローディング状態の管理を行うことができます。

```tsx
// 親コンポーネントで集権的にデータフェッチを管理
export default function DashboardPage() {
  const { data, isLoading } = useQuery({
    queryKey: ["dashboard"],
    queryFn: () => Promise.all([fetchUser(), fetchPosts(), fetchStats()]),
  });

  if (isLoading) return <Spinner />;

  const [user, posts, stats] = data;

  return (
    <main>
      <h1>Dashboard</h1>
      <UserProfile user={user} />
      <UserPosts posts={posts} />
      <UserStats stats={stats} />
    </main>
  );
}
```

しかし、このような実装では親から子、もしくは孫やその子孫コンポーネントまでデータを引き回す必要があります。このように、上位のコンポーネントで中央集権的にデータフェッチを管理してPropsを引き回すことはPropsの**バケツリレー**（Props Drilling）と呼ばれ、冗長で認知負荷が高い実装になるため、古くから忌避されてきました。

バケツリレーを避けようとすると末端でデータフェッチを行うこととなりますが、そのままローディング状態を管理するとポップコーンUIになってしまいます。

## Tanstack Queryでの考え方

Tanstack Queryにおいて、末端でデータフェッチを扱いつつ上位コンポーネントでローディング状態を管理する方法がいくつかあります。

### `useIsFetching()`

`useIsFetching()`は、Tanstack Queryがデータフェッチを行っているかどうかを管理するhooksです。これを利用すると、ページ全体でデータフェッチを行っているかどうかを管理することができます。

```tsx
function Dashboard() {
  const isFetching = useIsFetching();

  if (isFetching) return <Spinner />;

  return (
    <div>
      <UserProfile />
      <UserPosts />
      <UserStats />
    </div>
  );
}
```

ただし、これはTanstack Query固有のhooksであり、データフェッチを扱う他ライブラリに必ずしも用意されてるわけではありません。例えば[SWR](https://swr.vercel.app/ja)には同様のhooksは用意されていません。そのため、ライブラリによっては自作しないと末端のデータフェッチと統合的なローディング管理は難しい可能性があります。

また、このようなhooksの利用はあくまで開発者側が意図的にポップコーンUIを避ける意識が必要です。

### `<Suspense>`

[`<Suspense>`](https://ja.react.dev/reference/react/Suspense)はReactでデータフェッチを扱うための組み込みコンポーネントで、レンダリングの中断や再開、フォールバックUIなどを扱うことができます。`<Suspense>`を利用すると、開発者はローディング状態の管理をhooksベースではなく境界単位で考えることが強制されます。

:::message
より詳細に`<Suspense>`について知りたい方は、uhyoさんの[jotaiによるReact再入門 - Suspenseの基本](https://zenn.dev/uhyo/books/learn-react-with-jotai/viewer/suspense-basics)がわかりやすいと思うので、ご参照ください。
:::

Tanstack Queryにおいては、`useSuspenseQuery()`というhooksで`<Suspense>`とデータフェッチを統合することができます。

```tsx
export default function DashboardPage() {
  return (
    <main>
      <h1>Dashboard</h1>
      <Suspense fallback={<Spinner />}>
        <UserProfile />
        <UserPosts />
        <UserStats />
      </Suspense>
    </main>
  );
}

function UserProfile() {
  const { data } = useSuspenseQuery({
    queryKey: ["user"],
    queryFn: fetchUser,
  });
  return <div>{data.name}</div>;
}

// ...
```

![1つのSuspense境界でローディングを配置した場合](/images/popcorn-ui-anti-pattern/suspense-with-single-loader.png)

この場合、開発者（もしくはAI Agent）はローディング状態をどの単位で表示するか考えることになります。`useQuery()`ではコンポーネントに閉じるように実装しようとすると自然とポップコーンUIになってしまいましたが、`useSuspenseQuery()`を利用すると自然とSuspense境界やローディング体験を意識することになります。

このように、`<Suspense>`はローディング体験を統合的に考えやすい設計となっています。ただし、各コンポーネントを個別の`<Suspense>`で囲めばポップコーンUIと同じ結果になるので、乱用には注意が必要です。

## Next.jsでの考え方

この記事ではNext.jsを例に、メタフレームワークにおけるポップコーンUI対策について解説します。

[Next.js](https://nextjs.org/docs)は[React Server Components↗︎](https://ja.react.dev/learn/creating-a-react-app#which-features-make-up-the-react-teams-full-stack-architecture-vision)をサポートしているため、Server Componentsでより直感的にデータフェッチを扱うことができます。

```tsx
export default function Dashboard() {
  return (
    <div>
      <UserProfile />
      <UserPosts />
      <UserStats />
    </div>
  );
}

async function UserProfile() {
  const user = await fetchUser();
  return <div>{user.name}</div>;
}

async function UserPosts() {
  const posts = await fetchPosts();
  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}

async function UserStats() {
  const stats = await fetchStats();
  return <div>{stats.totalPosts} posts</div>;
}
```

上記のような実装でローディングを配置したい場合、Next.jsではRoute単位で`loading.tsx`を配置することで、統合的なローディング体験を実現することができます。

また、Streaming SSRなどもサポートしているため、重いデータフェッチを含むコンポーネントを`<Suspense>`境界単位で分割することなども可能です。

:::message
Next.jsにおける`<Suspense>`の活用については、[Next.jsの考え方 - SuspenseとStreaming](https://zenn.dev/akfm/books/nextjs-basic-principle/viewer/part_4_suspense_and_streaming)で詳しく解説しているので、ご参照ください。
:::

このように、Next.jsなどメタフレームワーク側でデータフェッチの統合的なローディング体験をサポートしていることは、ポップコーンUIを避けるための一助となります。

## 私見

Tanstack Queryなどのライブラリレベルでは、`useIsFetching()`や`useSuspenseQuery()`などのhooksを利用しつつ開発者が意図して実装することで、ポップコーンUIを防ぐことができます。しかし、`useQuery()`を用いて末端のコンポーネントでローディングをハンドリングすることは、コンポーネント単体で見れば自然な実装に見えるため、ポップコーンUIを防ぐには開発者がページ全体のローディング体験を意識する必要があります。

一方、`<Suspense>`は境界の実装であり、データを消費するコンポーネントの外でローディングの配置を考えることになります。hooksでは意図して防ぐ必要があるのに対し、`<Suspense>`は自然と考える余地が生まれるため、ポップコーンUIを避ける上でより適切な抽象化と考えられます。ただし、`<Suspense>`を乱用すればポップコーンUIになるのは同様なので、アンチパターンとしての認識は前提として重要です。

Next.jsなどメタフレームワークの規約に基づいたローディング体験は、開発者が意識せずとも統合的なローディングに導かれやすい設計になっています。AI Agent時代においては特に、開発者が意識すべきことが少ないことは大きなメリットとなるため、ポップコーンUIなどを防ぐ観点でもメタフレームワークを使うことの意義は大きいと考えられます。

## 考察

Reactは登場当初、設計思想として`UI = f(state)`という式でUIを表現していました。jQuery全盛の時代で、手続的なJavaScript実装が主流だった当時、この式は非常に革命的でした。この設計思想においてデータフェッチを統合するには、ローディングを状態として扱う必要があり、ポップコーンUI実装の構造的な原因となっていました。

Reactはその後、データフェッチをReactに統合する方向を模索し、その結果生まれたのが`<Suspense>`です。`<Suspense>`は`UI = f(state)`で表しきれなかったデータフェッチを補う抽象化であり、データフェッチを状態として扱うのではなく、レンダリングの中断と再開によって統合します。2025年のReact Confで発表された[Async React](https://www.youtube.com/watch?v=B_2E96URooA)では、`await UI = await f(await state)`という式が紹介されました。この式はUIそのものを非同期的なものとして再定義しており、昨今のReactの考え方を適切に表現していると考えられます。

このように、Reactはより良いUXを追求しており、その抽象化をReactに反映してきました。我々開発者は、ReactのAPIを学ぶだけではなく、Reactが抽象化しようとしてるより良いUXについても学ぶことが、AI Agent時代においてはより重要なのかもしれません。
