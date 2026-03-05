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

この記事ではポップコーンUIの実装について整理しつつ、ポップコーンUIを避けるための`<Suspense>`活用について考察します。

## ポップコーンUI

ポップコーンUIは冒頭述べたように、UIがポップコーンのようにランダムに置き換わっていくような体験のことを指します。これはパフォーマンス観点では、Core Web Vitalsの1つである[Cumulative Layout Shift (CLS)](https://web.dev/articles/cls?hl=ja)という指標で定量化して計測可能です。

TODO: gif - ポップコーンUIのデモ（スピナーがバラバラに解決される様子）

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

前述のTanstack Queryを使った実装例は、末端のコンポーネントでデータフェッチとローディングを扱ってるが故にポップコーンUIになります。親コンポーネントでこれらを中央集権的に管理すれば、統合的なローディング状態の管理を行うことができます。

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

### 統合的なローディング状態管理

Tanstack Queryにおいては、末端でデータフェッチを扱いつつ上位コンポーネントでローディング状態を管理するために、`useIsFetching()`というhooksが用意されています。

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

## `<Suspense>`

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

TODO: gif - Suspense版のデモ（境界単位で一括表示される様子）。ポップコーンUI版との対比

この場合、開発者（もしくはAI Agent）はローディング状態をどの単位で表示するか考えることになります。`useQuery()`ではコンポーネントに閉じるように実装しようとすると自然とポップコーンUIになってしまいましたが、`useSuspenseQuery()`を利用すると自然とSuspense境界やローディング体験を意識することになります。

このように、`<Suspense>`はローディング体験を統合的に考えやすい設計となっています。ただし、各コンポーネントを個別の`<Suspense>`で囲めばポップコーンUIと同じ結果になるので、乱用には注意が必要です。

## フレームワーク

### Next.js

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

Next.jsでは`loading.tsx`をRoute単位で配置可能で、統合的なローディング体験を実現することができます。Streaming SSRなどもサポートしているため、`<Suspense>`境界単位でchunkを分割することも可能です。

:::message
Next.jsにおける`<Suspense>`の活用については、[Next.jsの考え方 - SuspenseとStreaming](https://zenn.dev/akfm/books/nextjs-basic-principle/viewer/part_4_suspense_and_streaming)で詳しく解説しているので、ご参照ください。
:::

このように、Next.jsなどフレームワーク側でデータフェッチの統合的なローディング体験をサポートしてることは、ポップコーンUIを避けるための一助となります。

## 私見

- ポップコーンUIはReactの進化の文脈で捉えられる：hooksによる構造的な問題を、Suspenseという新たな抽象で解決した
- 従来: Reactは`UI = f(state)`と表現していた時期がある
  - この式にはLoadingやErrorといった非同期の状態が表現されていない
  - Suspenseにより、Loading体験の責務がコンポーネントから境界に移った
- Async React：`await UI = await f(await state)`（昨年のReact Confより）
- Reactの歴史はUIを非同期的なものとして捉え直し、その抽象を洗練させてきた歴史でもある
