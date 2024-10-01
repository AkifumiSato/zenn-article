---
title: "副作用のないレンダリング"
---

## 要約

Reactコンポーネントのレンダリングは副作用を含むべきでなく、これはServer Componentsにおいても同様です。

## 背景

Reactの最大の特徴の1つは、[宣言的](https://ja.wikipedia.org/wiki/%E5%AE%A3%E8%A8%80%E5%9E%8B%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0)であることです。従来、これは特に[宣言的UI](https://ja.react.dev/learn/reacting-to-input-with-state#how-declarative-ui-compares-to-imperative)という特徴として語られてきました。宣言的UIの前提条件として、Reactコンポーネントのレンダリングは副作用^[ここでは、他コンポーネントのレンダリングに影響しうる論理的状態の変更を指します]を持たず、純粋である必要があります。

https://ja.react.dev/learn/keeping-components-pure#side-effects-unintended-consequences

この強力な前提により、Reactはレンダリングを適切なタイミングで適切な範囲で行うことができます。

### Client Componentsにおける副作用

とはいえ、WebのUI実装にはイベントハンドラや非同期処理など、副作用がつきものです。Reactではこれらの副作用をイベントハンドラ内や`useEffect()`などの[Escape Hatches](https://ja.react.dev/learn/escape-hatches)で実装します。

https://ja.react.dev/learn/keeping-components-pure#where-you-\_can\_-cause-side-effects

## 設計・プラクティス

Server Componentsも同様にReactコンポーネントであり、レンダリングに副作用を含むべきではありません。App Routerもこの原則に沿って、各種APIが設計されています。

:::message
Server Componentsはページ遷移時に1度だけレンダリングされるように見えますが、実際にはServer Components内では`Promise`が`throw`されSuspendされることもあるため、レンダリングの中止と再開が行われる可能性があります。
:::

### Request Memoization、`React.cache()`

TBW

### 並行レンダリング

TBW

### Cookie操作

TBW

## トレードオフ
