---
title: "Request Memoization"
---

## 背景

## 設計・プラクティス

## 結論

## 例外

## memo

末端のコンポーネントでデータ取得を行うと重複するリクエストが多発するのではないかと懸念される方もいらっしゃると思います。App Routerではこのような重複リクエストを避けるため、[Request Memoization](https://nextjs.org/docs/app/building-your-application/caching#request-memoization)が実装されています。

```ts
async function getItem() {
  const res = await fetch("https://.../item/1");
  return res.json();
}

// <Component1 />
// <Component2 />
// の順で呼び出された場合

async function Component1() {
  const item = await getItem(); // cache MISS
  // ...
}

async function Component2() {
  const item = await getItem(); // cache HIT
  // ...
}
```

Request Memoizationはいわゆるメモ化なので、`fetch()`の引数に同じURLとオプション指定が必要です。上記例における`getItem()`のように、複数のコンポーネントで実行しうるデータ取得は関数などに抽出しておくことで、引数の一致ミスなど起こりづらくすることが望ましいでしょう。
