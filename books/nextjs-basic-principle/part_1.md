---
title: "第1部 データフェッチ"
---

Pages RouterからApp Routerへ移行する場合、[React Server Components](https://ja.react.dev/reference/rsc/server-components)をはじめ多くのパラダイムシフトを必要とします。データフェッチにおいては、Server Componentsによって従来よりセキュアでシンプルな実装が可能になった一方、使いこなすには従来とは全く異なる設計思想が求められます。

第1部では、App Routerのデータフェッチにまつわる基本的な考え方を解説します。

:::message
Next.jsにおけるデータアクセスのアプローチは、バックエンドAPIを分離して開発するか、Next.jsにデータベースアクセスを統合するかの2つに分けられます。本書で扱う「考え方」がこれらの選択によって大きく変わる部分は少ないと思われることもあり、本書では**バックエンドAPIを分離**するアプローチを前提に解説します。
:::
