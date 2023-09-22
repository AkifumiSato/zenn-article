---
title: "遷移と同期するstate管理ライブラリ、`location-state`"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "nextjs"]
published: false
---

## 構成

- 序文
  - `location-state`というライブラリの開発に携わらせてもらいました。
  - 先日、v1.0.0がリリースされました。
  - 本稿はこのライブラリの紹介記事です。
- モチベーション
  - Reactの状態管理については、様々な分類が可能です。
  - 筆者は過去に[スコープとライフタイムで考えるReact State再考](https://zenn.dev/terrierscript/articles/react-state-scope)という記事を書いており、ようやくするとstateにはスコープによる分類とライフタイム（stateの生存期間）による分類が可能であると考えています。
    - スコープによる分類で3種類
    - ライフタイムによる分類で6種類
  - state管理というと多くの場合スコープによる分類の話が多く、ライフタイムによる分類の話はあまり聞かない気がします。
  - SPA遷移時に状態が破棄され復元されない問題は[【翻訳】リッチなWebアプリケーションのための7つの原則](https://yosuke-furukawa.hatenablog.com/entry/2014/11/14/141415)など、2014年くらいにはすでに提唱されていますが、残念ながらその後もライフタイムに対する考慮はあまり広がっておらず、現在でもこの問題について対応されているSPAサイトは少なく感じるのが実情です。
  - この問題を解決すべく作られたのが[location-state](https://github.com/recruit-tech/location-state)です。
  - 実はこの問題に対する取り組みとして、筆者は過去に[recoil-sync-next](https://github.com/recruit-tech/recoil-sync-next)というライブラリ開発にも微力ながら携わらせていただきました。
  - しかしその後、recoilのメンテが滞り気味になってしまったことやrecoil・recoil-syncに依存した実装であることを避けたい気持ちが生まれ、本ライブラリの開発に至りました。
- 使い方
- 今後
- 感想
