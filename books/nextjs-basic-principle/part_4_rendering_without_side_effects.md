---
title: "副作用のないレンダリング"
---

## 要約

Reactのレンダリングは副作用を含むべきでなく、これはServer Componentsにおいても同様です。

## 背景

Reactの最大の特徴の1つは、[宣言的](https://ja.wikipedia.org/wiki/%E5%AE%A3%E8%A8%80%E5%9E%8B%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0)であることです。従来これは[宣言的UI](https://ja.react.dev/learn/reacting-to-input-with-state#how-declarative-ui-compares-to-imperative)という特徴として語られていました。

宣言的UIでは、UI（DOM）に対する直接的な操作を開発者が実装することはありません。副作用は分離され、コンポーネントの実行はReactによって適切なタイミングで適切な範囲で行われます。そのため、コンポーネントは高い冪等性が求められます。

### Client Componentsにおける副作用

WebのUI実装は副作用がつきものですが、Reactでは副作用を[Escape Hatches](https://ja.react.dev/learn/escape-hatches)に分離することで、コンポーネントを純粋に保つことができます。

- 従来からあるClient Componentsにおける副作用は、`useEffect`などのEscape Hatchesを利用することでzつ現可能に設計されていました
- Reactを従来より扱っていたならば、コンポーネントのコードにEscape Hatchesや3rd partyライブラリを利用せずに副作用を扱うことは御法度であることを知っているでしょう
- Server Componentsにおいても、これらは同様です

## 設計・プラクティス

- 序文
  - Server Componentsにおいても、レンダリングに副作用を含むべきではありません。
  - 副作用を含まないことが保証されることで、Reactはコンポーネントの実行を並行にしたり任意のタイミングで再実行したりすることができます
  - App Routerもこの原則に沿って、各種APIが設計されています。
- Request Memoization、`React.cache()`
- 並行レンダリング
- Cookie操作

## トレードオフ
