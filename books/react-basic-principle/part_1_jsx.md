---
title: "JSX"
---

## 要約

Reactは一般的にJSXで記述します。JSXは非常にシンプルなJavaScript拡張なため、JavaScriptの知識があれば実装可能でありながら、マークアップ構造をより直感的に記述することができます。

## 背景

Reactの関数コンポーネントでは、マークアップ構造を表現する`ReactElement`などを戻り値にする必要があります。これをJavaScriptで表現すると、以下のようになります。

```js
import React, { useState } from "react";

export function Counter() {
  const [count, setCount] = useState(0);

  return React.createElement(
    React.Fragment,
    null,
    React.createElement("p", null, "count: ", count),
    React.createElement(
      "button",
      { onClick: () => setCount(count + 1) },
      "increment",
    ),
  );
}
```

このコードから最終的に出力されるHTML構造を想像するのは、不可能ではないですが直感的ではありません。上記はシンプルな例なため、現実に扱うコンポーネントの場合には行数がさらに長くなり読みづらさを感じることでしょう。

## 設計・プラクティス

JSXはマークアップを直感的にJavaScriptで表現するための拡張構文です。前述の`Counter`コンポーネントをJSXで書き直すと、以下のようになります。

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

HTMLとは多少の違いはありますが、マークアップ構造が直感的に記述することが可能になります。

### 小さな拡張

JSXは、JavaScriptに対する拡張としては部分的で非常にシンプルです。いくつか[JSX固有のルール](#jsx固有のルール)はありますが、基本的にはHTMLやJavaScriptの知識で書き始めることができます。

JSXは小さな拡張でReactに依存しない仕様のため、[Vue](https://ja.vuejs.org/guide/extras/render-function)や[Solid.js](https://www.solidjs.com/tutorial/introduction_jsx)など様々なフレームワークでサポートされています。

## トレードオフ

### JSX固有のルール

JSXでは、以下のルールを守る必要があります。

- 単一のルート要素を返す
- すべてのタグを閉じる
- 基本的にキャメルケース

これらは、JSXがJavaScriptをシンプルに拡張する上で必要となるルールです。より詳細な内容について知りたい方は[公式ドキュメント](https://ja.react.dev/learn/writing-markup-with-jsx)を参照ください。

### トランスパイルは必須

JSXはそのままではブラウザは扱うことができないため、トランスパイルは必須です。初学者で特に強い理由がない場合、Reactでは[メタフレームワークの利用が推奨](https://ja.react.dev/learn/creating-a-react-app)されます。

もし自身でトランスパイル環境を構築する場合、様々なツールに関する知識が必要となります。
