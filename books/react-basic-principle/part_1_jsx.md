---
title: "JSX"
---

## 要約

Reactは一般的に[JSX](https://ja.react.dev/learn/writing-markup-with-jsx)で記述します。JSXは非常にシンプルなJavaScript拡張であるため学習コストは少なく、JavaScriptより直感的にマークアップ構造を記述することができます。

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

## 設計思想

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

多少違いはありますが、JSXはマークアップ構造をほとんどHTMLのように記述できます。一方、直接的にJavaScript変数や関数を埋め込めるので、HTMLとJavaScriptで分離してる時より振る舞いや依存関係が直感的に理解できる形にもなっています。

### 小さな拡張

JSXは、JavaScriptに対する拡張としては部分的で非常にシンプルです。後述するようにいくつか[JSX固有のルール](#jsx固有のルール)はありますが、基本的にはHTMLやJavaScriptの知識で書き始めることができます。

JSXは小さな拡張でReactに依存しない仕様のため、[Vue](https://ja.vuejs.org/guide/extras/render-function)や[Solid.js](https://www.solidjs.com/tutorial/introduction_jsx)など様々なフレームワークでサポートされています。

## トレードオフ

### JSX固有のルール

JSXでは、以下のルールを守る必要があります。

:::message
本書では概要の解説にとどまるので、より詳細な内容について知りたい方は[公式ドキュメント](https://ja.react.dev/learn/writing-markup-with-jsx)を参照ください。
:::

#### 単一のルート要素を返す

JSXでは必ず単一の要素を返すように実装する必要があります。ただし、Reactでは空のタグを表現するFragment（`<></>`）が存在するので、これを利用することでグループ化された要素を単一要素かのように表現することが可能です。

```jsx
// ✅: 全体をdivタグで囲んでいる
function ComponentA() {
  return (
    <div>
      <h1>タイトル</h1>
      <p>これはOKな例です。</p>
    </div>
  );
}

// ✅: 全体をフラグメント(<>...</>)で囲んでいる
//     不要なDOM要素を追加したくない場合に便利
function ComponentB() {
  return (
    <>
      <h1>タイトル</h1>
      <p>これもOKな例です。</p>
    </>
  );
}

// 🙅‍♂️: ルート要素がh1とpの2つ存在している
//     JSXでは、コンポーネントは必ず単一の要素（またはフラグメント）を返す必要があります。
/*
function ComponentC_NG() {
  return (
    // NG: このように並列に要素を返すことはできません
    <h1>タイトル</h1>
    <p>これはNGな例です。</p>
    // Error: JSX expressions must have one parent element.
  );
}
*/
```

#### すべてのタグを閉じる

JSXでは、一部HTMLタグで許容されている閉じタグの省略ができません。

```jsx
// ✅: imgタグとbrタグが自己終了タグ（ /> ）で閉じられている
function ComponentD() {
  return (
    <div>
      <p>画像を表示します:</p>
      <img src="image.jpg" alt="サンプル画像" />
      <br />
      <p>改行しました。</p>
    </div>
  );
}

// 🙅‍♂️: imgタグやbrタグが閉じられていない
//     HTMLでは許容される場合もありますが、JSXではエラーになります。
/*
function ComponentE_NG() {
  return (
    <div>
      <p>画像を表示します:</p>
      // NG: img タグが閉じられていない
      <img src="image.jpg" alt="サンプル画像">
      // NG: br タグが閉じられていない
      <br>
      <p>改行しました。</p>
      // Error: JSX elements must be closed, or be self-closing.
    </div>
  );
}
*/
```

#### 一部属性名の変更

JSXはJavaScriptに変換される都合上、`class`を`className`、`for`を`htmlFor`など独自の命名に変更してる場合があります。

```jsx
// ✅: labelタグの`for`は`htmlFor`で記述
<label htmlFor="my-input">ラベル</label>

// ✅: `class`は`className`で記述
<img
  src="https://i.imgur.com/yXOvdOSs.jpg"
  alt="Hedy Lamarr"
  className="photo"
/>
```

### トランスパイルは必須

JSXはそのままではブラウザは扱うことができないため、トランスパイルは必須です。初学者で特に強い理由がない場合、ReactではNext.jsなどの[メタフレームワークの利用が推奨](https://ja.react.dev/learn/creating-a-react-app)されます。

もし自身でトランスパイル環境を構築する場合、様々なツールに関する知識が必要となります。
