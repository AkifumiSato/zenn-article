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

## Suspense

- 概要：Reactが提供する非同期処理ハンドリングの仕組み
- アプローチ：Hooksでの状態管理から、Suspend（Promise throw）と`<Suspense>`境界によるUXの抽象化へ
- 子コンポーネントは`isLoading`を意識せず、データがある前提で書ける
- SuspenseはSuspense境界単位でのLoading解決=より統合的なLoading体験の抽象化
- デフォルトが統合的なLoading体験になる（`useIsFetching()`との対比：opt-in vs opt-out）

```tsx
// 子コンポーネントはローディング状態を扱わない
function UserProfile() {
  const { data } = useSuspenseQuery({
    queryKey: ["user"],
    queryFn: fetchUser,
  });
  return <div>{data.name}</div>;
}

function UserPosts() {
  const { data } = useSuspenseQuery({
    queryKey: ["posts"],
    queryFn: fetchPosts,
  });
  return (
    <ul>
      {data.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}

function UserStats() {
  const { data } = useSuspenseQuery({
    queryKey: ["stats"],
    queryFn: fetchStats,
  });
  return <div>{data.totalPosts} posts</div>;
}

// Suspense境界でLoading体験を一括管理
function Dashboard() {
  return (
    <Suspense fallback={<Spinner />}>
      <UserProfile />
      <UserPosts />
      <UserStats />
    </Suspense>
  );
}
```

- hooksベースとの本質的な違い：hooksは各コンポーネントがLoadingを自己解決する、Suspenseは境界がLoadingを代理解決する。Loading体験の責務の所在が異なる
- Suspense境界の設計指針：境界の粒度でUXをコントロールできる
- 注意：Suspenseも乱用すれば（全コンポーネントを個別のSuspense境界で囲むなど）ポップコーンUIになりうる。銀の弾丸ではないが、hooksベースに比べてLoading体験の単位が明示的なため、問題に意識が向きやすい

- TODO: gif - Suspense版のデモ（境界単位で一括表示される様子）。ポップコーンUI版との対比

### Waterfall（注意点）

- SuspenseでもWaterfallは起きうる（Suspense/hooks共通の注意点）
- ネストしたコンポーネントで順次fetchすると、親のSuspendが解決されるまで子のfetchが開始されない
- 対策：並列fetch（`Promise.all`、Tanstack Queryの`prefetchQuery`、Next.jsのServer Componentsでの並列fetchなど）

```tsx
// Tanstack Query: ルートやローダーで事前にfetchを開始する
async function dashboardLoader(queryClient: QueryClient) {
  await Promise.all([
    queryClient.prefetchQuery({ queryKey: ["user"], queryFn: fetchUser }),
    queryClient.prefetchQuery({ queryKey: ["posts"], queryFn: fetchPosts }),
    queryClient.prefetchQuery({ queryKey: ["stats"], queryFn: fetchStats }),
  ]);
}
```

```tsx
// Next.js Server Components: サーバー側で並列fetchする
async function Dashboard() {
  const [user, posts, stats] = await Promise.all([
    fetchUser(),
    fetchPosts(),
    fetchStats(),
  ]);

  return (
    <div>
      <UserProfile user={user} />
      <UserPosts posts={posts} />
      <UserStats stats={stats} />
    </div>
  );
}
```

## フレームワーク

### Next.js App Router

- Suspenseをフレームワークレベルで統合した例としてNext.js App Routerがある
- `loading.tsx`：ファイルを置くだけでルート単位のSuspense境界が設定される = デフォルトが統合的なLoading体験になる
- Server Components：サーバーサイドでデータを取得し、クライアントでのfetchライブラリ自体が不要になるケースもある
- フレームワークがSuspenseを前提に設計されていると、そもそもポップコーンUIが起きにくい構造になる

## 私見

- ポップコーンUIはReactの進化の文脈で捉えられる：hooksによる構造的な問題を、Suspenseという新たな抽象で解決した
- 従来: Reactは`UI = f(state)`と表現していた時期がある
  - この式にはLoadingやErrorといった非同期の状態が表現されていない
  - Suspenseにより、Loading体験の責務がコンポーネントから境界に移った
- Async React：`await UI = await f(await state)`（昨年のReact Confより）
- Reactの歴史はUIを非同期的なものとして捉え直し、その抽象を洗練させてきた歴史でもある
