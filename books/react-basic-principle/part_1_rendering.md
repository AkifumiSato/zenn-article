---
title: "レンダリングと宣言的UI"
---

## 要約

UIの更新処理は命令的な実装になりやすく、複雑化しがちです。Reactでは、状態の更新に応じてコンポーネントを**再レンダリング**することで、**宣言的UI**を可能にしています。

## 背景

Webアプリケーションをはじめ、近年のフロントエンド開発に対する要求は急速に複雑化しました。フレームワークなしの場合、開発者は以下のような様々な観点を考慮しつつ、UI更新処理を実装する必要があります。

- **更新前のUI**: DOMがどういう状態にあるか
- **アプリケーションの状態**: JavaScript変数やURLのクエリパラメータなど
- **イベント**: クリックや入力など
- **更新後のUI**: DOMをどういう状態にすべきか

これらを統合するUI更新処理は**命令的**になりやすく、命令的な実装はすぐに複雑化します。命令的な実装でも小さなサンプルであればうまくいくかもしれませんが、Webアプリケーションのように要求が複雑な場合には開発が困難になりがちです。

## 設計・プラクティス

Reactは状態の更新に応じてコンポーネントを**再レンダリング**することで、UIの更新処理を自動化し、**宣言的UI**を可能にしています^[参考: [宣言型 UI と命令型 UI の比較](https://ja.react.dev/learn/reacting-to-input-with-state#how-declarative-ui-compares-to-imperative)]。

### Reactのメンタルモデル

Reactは、`UI = f(state)`のように**UIを関数と状態で表現できる**と考えました。`f()`はUIを表現するコンポーネント関数で、`state`は文字通り状態です。状態の更新に応じて`f(state)`を再実行し反映すれば、常にあるべきUIを表現できます。

このように、状態の更新に応じて`f(state)`を再実行することをReactでは**再レンダリング**と呼びます^[参考: [レンダリング大全](https://zenn.dev/txxm/articles/f04b21949ddab3)]。また、`UI = f(state)`のように「欲しい結果」に主眼を置いて実装するようなプログラミングスタイルは宣言的プログラミングと呼ばれるため、Reactのアプローチは**宣言的UI**と呼ばれています。

::::details `UI = f(state)`とReact Server Components
:::message
React Server Componentsによって、上述のメンタルモデルは更新され、以下の2つが統合される形となりました。

- `UI = f(state)`: Client Components
- `UI = f(data)`: Server Components

詳細な解説はDan Abramov氏の[The Two Reacts](https://overreacted.io/the-two-reacts/)を参照ください。
:::
::::

### 再レンダリングの大まかな仕組み

より具体的には、再レンダリングは`useState()`から返されるsetter関数を呼び出すことでトリガーされます。

```tsx
export function Counter() {
  const [count, setCount] = useState(0);

  return (
    <>
      <p>count: {count}</p>
      <button onClick={() => setCount(count + 1)}>increment</button>
    </>
  );
}
```

上記の例では、クリックのたびに`setCount`が呼び出され、`<Counter>`が再レンダリングされます。

Reactは「更新前のUI」と「更新後のUI」を比較して、差分があった場合のみ実際のDOMに変更を反映します^[参考: [レンダーとコミット
](https://ja.react.dev/learn/render-and-commit)]。そのため、再レンダリングの結果差分がない場合にはDOMへの適用はスキップされます。

## トレードオフ

### 副作用の取り扱い

宣言的であることは、可読性や堅牢性など様々な観点でメリットが得られます。一方、現実のコンポーネントにはブラウザのAPIとの疎通や外部APIへのデータフェッチなど、何かしらの副作用が含まれることが想定されます。

Reactは、これらに対し[Escape Hatches](https://ja.react.dev/learn/escape-hatches)として、`useEffect()`や`useRef()`など副作用を扱うためのhooksを用意しています。これらは文字通りできるだけ避けるべき避難口ではありますが、現実的には必要不可欠なものでもあります。

### 再レンダリング範囲の適正化

Reactは状態が更新されたコンポーネント自身と、その子孫コンポーネントを再帰的にレンダリングします。そのため、上位のコンポーネントで再レンダリングを頻繁にトリガーすると、パフォーマンス的に不利になりえます。

![React Tree](/images/react-basic-principle/re-render-tree.png)

状態を上位のコンポーネントで持たざるを得ない場合には、不要なレンダリングをスキップするために[`memo`](https://ja.react.dev/reference/react/memo)の使用を検討してください。
