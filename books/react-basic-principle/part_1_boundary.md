---
title: "レンダリングの境界"
---

## 要約

Reactでは`<Suspense>`や`"use client"`など、様々なレンダリングの**境界**を定義できます。レンダリングの境界はUIをツリー構造で考えるコンポーネント指向に組み込みやすく、ツリー構造と相性が悪い最適化や前提条件の組み込みで境界の概念が採用されています。

## 背景

Reactをはじめ、コンポーネント指向ではUIをツリー構造で考えます。ツリー構造は情報の依存関係が明確になるため、全体の構造把握が容易になります。

<!-- https://excalidraw.com/#json=1BqNUBkEjbP4Napz-xkwR,tYubY5DXn_2a45eDcUngwQ -->

![UI Tree](/images/react-basic-principle/ui-tree.png)

ツリー構造は、コンポーネントを組み合わせたマークアップ構造として表現されます。

```tsx
function Page() {
  const { data } = usePageContents();

  return (
    <Layout>
      <Header />
      <SideNav />
      <MainContents>
        <Article data={data} />
        <InquiryForm />
      </MainContents>
    </Layout>
  );
}
```

一方、UIに関するあらゆる実装要件がツリー構造とマッチするわけではありません。例えば、末端のコンポーネントでエラーが発生した場合に、祖先コンポーネントでエラーUIを表示したいとします。この際、ツリー構造のルールに厳密に従うにはエラーを`throw`するのではなく、エラーハンドリング関数などを*Props Drilling*（Propsのバケツリレー）することが考えられますが、これは冗長な実装が伴うため良い設計ではありません。

![Error UI](/images/react-basic-principle/error-props-dlilling.png =350x)

## 設計思想

Reactでは、様々な課題解決のためにレンダリングに**境界**（Boundary）を定義するアプローチが採用されています。レンダリングの境界は、UIをツリー構造で考えるコンポーネント指向のメンタルモデルに組み込みやすいため、ツリー構造な設計だけでは解決が難しい場合に、Reactから提供される解決策として採用されることがあります。

![Boundary](/images/react-basic-principle/boundary.png =350x)

レンダリングの境界はそれぞれ目的や定義の仕方が異なりますが、大別するとコンポーネントかディレクティブによって定義できます。本書では以下4つの境界について解説します。

- [`<ErrorBoundary>`](#errorboundary)
- [`<Suspense>`](#suspense)
- [`"use client"`](#use-client)
- [Context](#context)

:::message
レンダリングの境界には、Reactではなくフレームワークによって定義されているものも存在します。例えば、Next.jsの[`"use cache"`](https://nextjs.org/docs/app/api-reference/directives/use-cache)は、関数やコンポーネントに対するキャッシュの境界を定義するのに用いられます。
:::

### `<ErrorBoundary>`

`<ErrorBoundary>`は、レンダリング中にエラーが発生した場合にfallbackを表示するための境界を定義するコンポーネントです。言い換えると、`<ErrorBoundary>`はアプリケーション全体がクラッシュするのを防ぎ、部分的にエラーUIを表示します。

```tsx
<ErrorBoundary fallback={<ErrorDialog>error message</ErrorDialog>}>
  <SomeComponent />
</ErrorBoundary>
```

`<ErrorBoundary>`は他の境界とは異なり、開発者が自身で内容を実装できる境界コンポーネントです。ただし、執筆時現在では歴史的経緯により[Classコンポーネント](https://ja.react.dev/reference/react/Component#defining-a-class-component)でのみ実装することが可能です。

:::details `<ErrorBoundary>`の実装例

```jsx
// https://ja.react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    // Update state so the next render will show the fallback UI.
    return { hasError: true };
  }

  componentDidCatch(error, info) {
    logErrorToMyService(
      error,
      // Example "componentStack":
      //   in ComponentThatThrows (created by App)
      //   in ErrorBoundary (created by App)
      //   in div (created by App)
      //   in App
      info.componentStack,
      // Only available in react@canary.
      // Warning: Owner Stack is not available in production.
      React.captureOwnerStack(),
    );
  }

  render() {
    if (this.state.hasError) {
      // You can render any custom fallback UI
      return this.props.fallback;
    }

    return this.props.children;
  }
}
```

:::

:::message
筆者は[react-error-boundary](https://www.npmjs.com/package/react-error-boundary)を好んで利用します。
:::

### `<Suspense>`

`<Suspense>`は、レンダリング中に発生した**サスペンド**に対し、fallbackを表示することができる組み込みのコンポーネントです。サスペンドとは、レンダリング中に`Promise`を`throw`することを意味します。サスペンドが発生すると、`<Suspense>`コンポーネントはpropsで受け取った`fallback`を表示します。

これにより、非同期処理の待機時UI表示が容易に実装できます。

```tsx
<Suspense fallback={<Loading />}>
  <SomeComponent />
</Suspense>
```

`<Suspense>`はよくデータフェッチと組み合わせて、読み込み時のUIを記述するのに利用されます。`<Suspense>`は組み込みコンポーネントのため、ライブラリやフレームワークがサスペンドするのであれば、書き方は常に一定になります^[開発者が自前でサスペンドするような実装は推奨されません]。

より`<Suspense>`について詳しく知りたい方は、[uhyoさん](https://x.com/uhyo_)の以下の本がとてもわかりやすいのでぜひご一読ください。

https://zenn.dev/uhyo/books/react-concurrent-handson

### `"use client"`

`"use client"`は、Reactの新たなアーキテクチャと仕様である[React Server Components](https://ja.react.dev/reference/rsc/server-components)の一貫で追加された、新たなディレクティブです。[Next.js](https://nextjs.org/)ではコンポーネントのデフォルトはServre Componentsとなり、サーバー側でのみレンダリングされます。クライアント側でもレンダリングするには、`"use client"`で**Client境界**（Client Componentsの境界）^[`"use client"`は境界となるコンポーネントを含むファイルの先頭で宣言する必要があります。]を明示的に指定し、オプトインする必要があります。

```tsx
"use client";

export function SomeComponent() {
  // ...
}
```

:::message
Client Componentsには`children`などを通してServer Componentsを渡すことができます。詳しくは[「Next.jsの考え方 - Compositionパターン」](https://zenn.dev/akfm/books/nextjs-basic-principle/viewer/part_2_composition_pattern)を参照ください。
:::

### Context

[Context](https://ja.react.dev/learn/passing-data-deeply-with-context)はデータや関数を共有する境界で、`createContext()`によって作成できるコンポーネントです。Context境界内でデータや関数を共有することで、*Props Drilling*を避けることができます。

```tsx
const SomeContext = createContext(defaultValue);

// ...

<SomeContext value={myContextValue}>
  <SomeComponent />
</SomeContext>;
```

多くの3rd partyライブラリでは、*Provider*という名称でContext境界のコンポーネントを提供しています。

:::message
従来Contextの境界は`<MyContext.Provider>`とする必要がありましたが、[React19](https://ja.react.dev/blog/2024/12/05/react-19#context-as-a-provider)以降は`createContext()`の結果をそのままコンポーネントとして利用できるようになりました。
:::

## トレードオフ

### 見えない境界

Propsとは異なり、レンダリングの境界は見えない前提条件です。境界は必ずしもコンポーネント宣言の近くにあるとは限らないため、実装時にはコード上に見えない前提条件への配慮が必要な場合があります。

- `"use client"`が遠い祖先要素にあり、末端コンポーネントの実装からはClient Componentsかどうか判断できない
- テストを書いてみたら依存してるContext境界を定義してなかったためにエラー
