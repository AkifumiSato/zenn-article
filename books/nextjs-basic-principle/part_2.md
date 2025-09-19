---
title: "第2部 コンポーネント設計"
---

[RSCのRFC↗︎](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md)には以下のような記述があります。

> The fundamental challenge was that React apps were client-centric and weren’t taking sufficient advantage of the server.
> _根本的な課題は、Reactアプリがクライアント中心で、サーバーを十分に活用していないことだった。_

Reactコアチームは、Reactが抱えていたいくつかの課題を個別の課題としてではなく、根本的には**サーバーを活用できていない**ことが課題であると考えました。そして、サーバー活用と従来のクライアント主体なReactの統合を目指し、設計されたアーキテクチャがReact Server Componentsです。

[第1部 データフェッチ](part_1)で解説してきた通り、特にデータフェッチに関しては従来よりシンプルでセキュアに実装できるようになったことで、ほとんどトレードオフなくコンポーネントにカプセル化することが可能となりました。一方コンポーネント設計においては、従来のクライアント主体のReactコンポーネント相当であるClient Componentsと、Server Componentsをうまく統合していく必要があります。

第2部ではRSCにおけるコンポーネント設計パターンを解説します。
