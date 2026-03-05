---
title: "ポップコーンUIとReact"
emoji: "🍿"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "ui"]
published: false
---

- 現象の共有：複数APIを叩くページで、スピナーがバラバラと消えてレイアウトがガタつく体験
- 命名：この現象を「ポップコーンUI」と呼ぶ（ポップコーンが弾けるように順次表示される様子）
- 構造的な問題：hooksベースのデータフェッチを素朴に使うと自然に発生する
  - AI Agentによるコード生成でもこの傾向は顕在化しやすい
- Reactはより良いUXの抽象化を追求してきており、`<Suspense>`はその成果
- 本記事の目的：ポップコーンUI問題の整理と、Reactがこの問題にどう向き合ってきたかの解説

## ポップコーンUI

- 定義：ポップコーンのように、複数のスピナーがランダムに置き換わっていくような体験
- 影響：Layout Shiftの発生=CLS悪化

:::details Cumulative Layout Shift（CLS）とは
- Webパフォーマンスの潮流：ユーザー体験（UX）の定量化
- Core Web Vitals：UX定量化のための重要指標群
- CLS：予期せぬレイアウト変化（ガタつき）による不快感の計測
:::

- Tanstack Queryの例：`useQuery`によるローディング状態（`isLoading`）の取得をもとに、バケツリレーを避けて実装すると自然とポップコーンUIになる

:::details Propsのバケツリレー（Props Drilling）とは
- 定義：上位でのデータフェッチと末端へのProps引き回し
- 歴史：React初期の忌避パターンで、Redux等により解決されていた
- 再燃：Next.js（Pages Router）の`getInitialProps`や`getServerSideProps`などにより、バケツリレー設計に回帰した
- 反動：バケツリレーへの忌避から、各コンポーネントが個別にデータフェッチする設計が広まった
:::
  - hooksベースでデータフェッチを扱うと、コンポーネント単位でのLoading解決に目が向きがち
  - 統合的なLoading体験を実装するには上位で管理しバケツリレーするなど、工夫が必要
- 構造的ジレンマ：hooksベースのAPIをもとに自然な実装をしようとすると、ポップコーンUIになる

- TODO: gif - ポップコーンUIのデモ（スピナーがバラバラに解決される様子）

```tsx
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

// 親はただ並べるだけ → 3つのスピナーがバラバラに解決される
function Dashboard() {
  return (
    <div>
      <UserProfile />
      <UserPosts />
      <UserStats />
    </div>
  );
}
```

### エコシステム内での対処：useIsFetching()

- Tanstack Queryでは`useIsFetching()`で上位のコンポーネントでLoading状態を管理することができる
- 問題点：
  - デフォルトがポップコーンUIのままであり、意図して避けないといけない
  - 子コンポーネント側の`isLoading`分岐が残りうるので、二重管理になりがち
- 位置づけ：hooksベースのエコシステム内での対症療法であり、設計レベルの解決ではない

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
