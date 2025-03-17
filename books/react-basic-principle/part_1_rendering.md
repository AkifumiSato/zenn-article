---
title: "レンダリングと宣言的UI"
---

## 要約

UIの更新処理は命令的になりやすく、命令的な実装は複雑化しがちです。Reactでは、状態の更新に応じてコンポーネントを**再レンダリング**することで、**宣言的UI**を可能にしています。

## 背景

Webアプリケーションの台頭により、近年のフロントエンド開発に対する要求は急速に複雑化しました。フレームワークなしの場合、開発者は以下のような様々な観点を考慮しつつUI更新処理を実装する必要があります。

- **更新前のUI**: DOMがどういう状態にあるか
- **JavaScript管理の状態**: JavaScript変数など
- **イベント**: クリックや入力など
- **更新後のUI**: DOMをどういう状態にすべきか

これらを統合するUI更新処理は**命令的**になりやすく、命令的な実装はすぐに複雑化します。命令的な実装でも小さなサンプルであればうまくいくかもしれませんが、Webアプリケーションのように要求が複雑な場合には開発が困難になりえます。

## 設計・プラクティス

Reactは状態の更新に応じてコンポーネントを**再レンダリング**することで、**宣言的UI**を可能にしています。言い換えると、ReactはUI更新処理を自動化し、命令的な実装を排除しています^[参考: [宣言型 UI と命令型 UI の比較](https://ja.react.dev/learn/reacting-to-input-with-state#how-declarative-ui-compares-to-imperative)]。

### Reactのメンタルモデル

Reactは命令的実装を排除するために、「更新後のUI」を`UI = f(state)`のように関数と状態で表現できると考えました。`f()`はUIを表現する関数で、`state`は文字通り状態です。状態の更新に応じて`f(state)`を再実行し反映すれば、「更新後のUI」を実現できます。

このように、状態の更新に応じて`f(state)`を再実行することをReactでは**再レンダリング**と呼びます^[参考: [レンダリング大全](https://zenn.dev/txxm/articles/f04b21949ddab3)]。また、`UI = f(state)`のように欲しい結果を書くようなプログラミングスタイルは宣言的プログラミングと呼ばれるため、Reactのアプローチは**宣言的UI**と呼ばれています。

:::message
React Server Componentsによって、上述のメンタルモデルは更新され、以下の2つが統合される形となりました。

- `UI = f(state)`: クライアントサイド
- `UI = f(data)`: サーバーサイドのみ

詳細な解説はDan Abramov氏の[The Two Reacts](https://overreacted.io/the-two-reacts/)を参照ください。
:::

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

Reactは「更新前のUI」と「更新後のUI」を比較して、差分があった時のみ実際にDOMに差分情報を反映します。そのため、再レンダリングされた結果が全く一緒の場合には適用がスキップされます。

:::message
これらはレンダーとコミットと呼ばれる処理です。より詳細な内容を知りたい方は[公式ドキュメントの解説](https://ja.react.dev/learn/render-and-commit)をご参照ください。
:::

## トレードオフ

### 副作用の取り扱い

宣言的であることは、可読性や堅牢性など様々な観点でメリットが得られます。一方、現実のコンポーネントには何かしらの副作用が含まれることが想定されます。先述の「JavaScript管理の状態」も、副作用の1つです。他にも、ブラウザのAPIとの疎通などフロントエンド開発は副作用で溢れています。

Reactは、これらに対し[Escape Hatches](https://ja.react.dev/learn/escape-hatches)として、`useEffect()`や`useRef()`など副作用を扱うためのhooksを用意しています。これらは文字通りできるだけ避けるべき手段ではありますが、現実的には必要不可欠なものでもあります。

また、最も重要な副作用とも言えるデータフェッチについては、React Server Components以降ではServer Componentsで行うことが推奨されます。これは前述の[The Two Reacts](https://overreacted.io/the-two-reacts/)にも記述されているので、ご参照ください。

### 再レンダリング範囲の適正化

Reactは状態の更新があったコンポーネントに含まれるツリー全体を再レンダリングします。そのため、上位のコンポーネントで再レンダリングを頻繁にトリガーすると、パフォーマンス的に不利になりえます。

![React Tree](/images/react-basic-principle//re-render-tree.png)

上位のコンポーネントで状態を持たざるを得ない場合には、不要なレンダリングをスキップするために[`memo`](https://ja.react.dev/reference/react/memo)の使用を検討するなどしましょう。
