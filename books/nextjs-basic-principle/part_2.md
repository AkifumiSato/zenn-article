---
title: "第2部 コンポーネント設計"
---

第1部の[データフェッチ on Server Components](part_1_server_components)でも述べた通り、React Server Componentsは**Reactがよりサーバーを活用できること**を目指して生まれたアーキテクチャです。これを踏まえたコンポーネント設計は従来とは大きく異なるものとなります。

第1部で述べたデータフェッチ設計はコンポーネント設計に大きく影響しますが、Server Componentsにはバンドルサイズや効率的なレンダリングなど他にも様々なメリットがあり、コンポーネント設計はこれらも加味した上で決定する必要があります。また、React Server ComponentsにおいてはClient ComponentsがServer Componentsを`import`してはならないという大きな制約が存在します。

第2部ではこういったメリットやルールも加味した上で、App Routerにおけるコンポーネント設計パターンを解説します。
