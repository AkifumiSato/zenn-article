---
title: "Reactチームが見てる世界、Reactユーザーが見てる世界"
emoji: "🌏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: false
---

:::message
この記事は、[React Tokyo#2](https://react-tokyo.connpass.com/event/343757/)で公演した[スライド](https://akifumisato.github.io/slide-of-react-tokyo-202502/1)を元に執筆したものです。
:::

Reactは小規模でシンプルなサイトから複雑で大規模なアプリケーションまで、非常に幅広く採用されている人気のフレームワークです。OSS化から10年以上の歴史もありながら、昨今もReact Server Componentsなど革新的な進化を我々に提案し続けています。

一方で、React Server Componentsへの批判的意見やReact19 RCで判明した[Boomer fetching問題](https://github.com/facebook/react/issues/29898)を見ていると、Reactチームと多くのReactユーザーの間に多くの齟齬があるように感じました。

- 序文
  - 昨今RSCへの批判的意見やBoomer fetchingの炎上など、
  - Reactの歴史を振り返ると、一貫してシンプルな解決策を出してきたように見えるが、どこかに
  - 昨今ReactチームとReactユーザーの間で多くの齟齬があることに気づきました
  - この記事を通じて、Reactチームのコンテキストや、Reactの意思決定について読者の理解を深められたら幸いです。
- 概要
  - Metaは非常に大規模な開発体制で、フレームワーク開発も厭わない
  - 一方大多数のReactユーザーは、小規模〜中規模な開発でReactを利用している
  - Reactは大規模開発を支える技術でありながら、小規模〜中規模でも通用する技術である
    - Reactは低レイヤーな技術と言える
    - 大規模な開発を支える低レイヤーな技術は、小規模〜中規模でも通用しやすい
    - 一方小規模〜中規模に主眼を置いた技術は、大規模でも通用するとは限らない
  - Reactは大多数のReactユーザーのためにReactを開発してるのではない。大規模開発を支える技術として、Reactを開発している
  - Reactの意思決定に対する批判の多くは、見てる世界の違いが原因なことが多いように感じる
  - 議論も意思決定も、Reactチームの目線を理解することが重要だと、筆者は考える
- Metaの大規模開発
  - 世界中に独自のデータセンターを配置
  - フレームワークや言語開発も厭わない開発予算
  - 現在では、Meta社内には数万のコンポーネントが存在
    - 10万以上という噂も
    - 多くの修正を自動化しているらしい（Code mod？）
- Reactの誕生
  - 2012年のWeb開発
  - 2012年のMeta
  - bolt.js
  - React
  - ReactのOSS化
  - Reactの拡大
- ReactとGraphQL
  - GraphQLの誕生と拡大
  - MetaにとってのGraphQL
  - MetaにとってのReactとGraphQL
- React Server Components
  - 2020年頃、Reactが抱えていた課題
  - React Server Componentsの誕生
  - GraphQLとRSCの自立分散性
- Reactチームが見てる世界、Reactユーザーが見てる世界
  - Reactは大規模開発を支える技術
  - 大多数のReactユーザーは小規模〜中規模開発でReactを採用してる
- まとめ
  - Reactは大規模開発を支えるシンプルなフレームワークである
  - MetaはReact+GraphQLで自立分散的なアーキテクチャを採用している
  - React Server Componentsもまた自立分散的アーキテクチャであり、正常進化である
