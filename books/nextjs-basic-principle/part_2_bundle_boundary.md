---
title: "クライアントとサーバーのバンドル境界"
---

## 要約

`"use client"`や`"use server"`は実行環境を示すものではありません。これらはバンドラにバンドル境界を宣言するためのものです。実行環境に制約を儲けたい場合は`client-only`や`server-only`を使いましょう。

## 背景

- RSCではServerとClient2つのバンドルが出てくる
  - だからバンドラ側の対応が必須
  - RSCとしてのルールが存在する
- RSCのルールである`"use client"`や`"use server"`が「実行環境をマークするためのもの」と誤解されることがよくある
  - これは事実ではない

## 設計・プラクティス

`"use client"`や`"use server"`はClientバンドルやServerバンドルの境界（Boundary）を宣言するためのものである

![境界とディレクティブ](/images/nextjs-basic-principle/rsc-layer.png)

- `"use client"`と`"use server"`はバンドル境界の起点
  - Dan先生風にいうならば「2つの世界、2つのドア」
- 実行環境の制約
  - `client-only`や`server-only`で実行環境の制約を行うことができる
- アイランドアーキテクチャとの類似性
  - アイランドアーキテクチャも同様にServerバンドルにClientバンドルをOptinできる設計
  - 古くはPHP+jQueryも同様の2層アーキテクチャ

## トレードオフ

- Client BoundaryではServer Componentsを読み込めない
- ファイル単位の`"use server"`には注意が必要
  - ["use server"; でexportした関数が意図せず？公開される](https://zenn.dev/moozaru/articles/b0ef001e20baaf)参照
