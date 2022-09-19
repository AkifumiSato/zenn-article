---
title: "feature sliced design:深化するフロントエンドアーキテクチャ"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "vue"]
published: false
---

## feature sliced designとは

[**feature sliced design**](https://feature-sliced.design/)（以下FSD）とは、フロントエンドのディレクトリ設計方法の1つです。公式には「Architectural methodology for frontend projects(フロントエンドのアーキテクチャ方法論)」とされています。

FSDは以下のようなLayers・Slices・Segmentsといった3つの概念が登場し、以下のような構成になります。

![schema](/images/feature-sliced-design/schema.png)

この構成はスコープとビジネスドメインに強くフォーカスしており、プロジェクトがスケールするにつれて発生するよくある問題の一部、例えば増えすぎたコンポーネントによる複雑な依存関係が引き起こす問題などを防止してくれます。また、FSDはこの構成の不完全さについても言及しており、小規模すぎる場合や大規模すぎる場合にはデメリットの方が上回ってしまう可能性を強調しています。FSDは非常によく研究されており、段階的導入も可能なので、一定規模で複雑性が高くなったプロジェクトやそれが見込まれるプロジェクトにおいては一考の価値があると思います。

:::message
本稿は、FSDのv2について筆者なりにポイントをまとめたものになります。筆者はFSDはまだ学んでいる途中なので、記事の内容に誤りがあった場合はご指摘ください。
:::

## 目的

- 目的
- コンセプト
- 概要
    - Layers・Slices・Segments
    - 依存関係のルール
        - https://feature-sliced.design/docs/concepts/app-splitting#layers-order
- 各Layers
- Slices（ドメイン）
- Segments
- 導入
- 参考

https://profy.dev/article/react-folder-structure
https://feature-sliced.design/docs/get-started/quick-start
