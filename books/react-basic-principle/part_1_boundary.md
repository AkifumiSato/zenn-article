---
title: "レンダリングの境界"
---

## 要約

Reactでは`<Suspense>`や`"use client"`など、様々な形で**レンダリングの境界**を定義できます。レンダリングの境界は多様な前提条件を作り込める一方、UIをツリー構造で考えるコンポーネント指向に組み込みやすいため、Reactでは様々な課題解決のために境界の概念が採用されています。

## 背景

TBW

- Reactをはじめ、コンポーネント指向ではUIをツリー構造で考えて設計します。
- ツリー構造では親子関係で物事を考えていくため、子孫や祖先の存在は基本的に意識しません。
- 一方、現実のUI実装ではエラーハンドリングや並行処理、サーバーとクライアントなど様々なケースで親子関係を超えて、子孫や祖先の影響を考慮しないと設計がしづらいケースがあります。

## 設計思想

- Reactは、UIを表現するツリー構造に対し**レンダリングの境界**を定義することで、様々な前提を作り込むアプローチを採用しています。
- レンダリング境界はコンポーネント指向のUIをツリー構造で考えるメンタルモデルに組み込みやすいため、Reactでは様々な課題解決のために境界の概念が採用されています。

![Boundary](/images/react-basic-principle/boundary.png =350x)

- レンダリングの境界は、コンポーネントやディレクティブによって定義できます。以下は、Reactが提供する代表的な境界定義です。

- `<ErrorBoundary>`
- `<Suspense>`
- `"use client"`
- Provider

### `<ErrorBoundary>`

TBW

- `<ErrorBoundary>`はレンダリング中にエラーが発生した場合にfallbackを表示するための境界を定義するコンポーネントです。
- `<ErrorBoundary>`は他の境界とは異なり、開発者が自身で内容を実装できる境界コンポーネントですが、古くから存在していることもありClassコンポーネントで記述する必要があります。

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

プロジェクトごとに毎度実装するのも手間なので、筆者は[react-error-boundary](https://www.npmjs.com/package/react-error-boundary)を利用することを好んでいます。

### `<Suspense>`

TBW

- `<Suspense>`はレンダリング中に発生した**サスペンド**に対し、fallbackを表示することができる組み込みのコンポーネントです。
- サスペンドとは、レンダリング中に`Promise`を`throw`することを意味します。サスペンドが発生すると、`<Suspense>`コンポーネントはpropsで受け取った`fallback`を表示します。
- 通常開発者が自身でサスペンドするような実装は推奨されませんが、ライブラリやフレームワーク側でサスペンドすることでデータフェッチとローディングの表示などの書き方に一貫性が得られます。
- より詳しくは、uhyoさんの以下の本がとてもわかりやすいので、呼んだことがない方はぜひご一読ください。

https://zenn.dev/uhyo/books/react-concurrent-handson

### `"use client"`

TBW

- `"use client"`は、Reactの新たなアーキテクチャと仕様である[React Server Components](https://ja.react.dev/reference/rsc/server-components)の一貫で追加された、新たなディレクティブです。
- [Next.js](https://nextjs.org/)などではデフォルトでServre Componentsとなるため、Client境界（Client Componentsの境界）^[`"use client"`は境界となるコンポーネントを含むファイルの先頭で宣言する必要があります。]で`"use client"`を宣言する必要があります。

:::message
Client Componentsには`children`などを通してServer Componentsを渡すことができます。詳しくは[「Next.jsの考え方 - Compositionパターン」](https://zenn.dev/akfm/books/nextjs-basic-principle/viewer/part_2_composition_pattern)を参照ください。
:::

### Context（Provider）

TBW

- [Context](https://ja.react.dev/learn/passing-data-deeply-with-context)は情報を共有する境界です。
- Contextを通して情報を境界内で共有することで、_Props Drilling_（Propsのバケツリレー）を避けることができます。

:::message
[React19](https://ja.react.dev/blog/2024/12/05/react-19#context-as-a-provider)以前は`createContext()`の結果をそのままコンポーネントとして利用できるようになりましたが、従来は`<MyContext.Provider>`のように境界コンポーネントが提供されていました。そのため、本書ではContext（Provider）と表現しています。
:::

## トレードオフ

### 見えない境界

TBW

- コンポーネントにPropsを渡すのとは異なり、レンダリングの境界は必ずしもコンポーネント宣言の近くにあるとは限りません。
- 特に、再利用性の高いコンポーネントや深い階層のコンポーネントを実装している時には境界を意識しづらいことがあります。
- 境界が意識しづらいことが問題になるとすれば、`"use client"`によるClient境界が利用者の祖先側にあった場合でしょう。
- Next.jsでは、Client境界外でClient Componentsでのみ利用可能なAPIを利用した場合にはエラーと修正のヒントが明示されます。
