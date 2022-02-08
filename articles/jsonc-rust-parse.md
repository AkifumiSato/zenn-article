---
title: "Rustでパーサーの基礎を学ぶ"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
---

## 構成

- Introduction
  - トランスパイル、コンパイル当たり前な時代
  - 実際なにやってるんだろうってところが敷居高いよね
  - JSON 書いてる人おるな
  - やってみるか
  - 同じじゃつまらんから JSONC にしよう
- 作るもの
  - JSONC とは
  - JSONC を解析する wasm
  - wasm の戻り値は string で
- rust tips
  - 基本自分の理解のためなので解析系ライブラリはあんま使わない
  - anyhow
  - thiserror
- 解析の流れ
  - 字句解析
    - token
  - 構文解析
    - AST,Node
    - 再帰下降分析
  - AST 元に組み立て
- 字句解析
  - lexer を作る
  - 1 文字で完結する場合
  - Number
  - 文字列
  - Null、Boolean
  - Object、配列
- 構文解析
  - token を流して tree 化していく
  - 予期せぬ token が来たらちゃんとわかりやすくエラーメッセージつけよう
- JSON string を返す
  - wasm の入り口を用意
  - Node を stringfy
  - 呼び出せば OK
- 感想
  - 全体的に手続き的になってしまった
  - コンビネータというのを後で知った
  - エラー雑に実装したけどよくない
  - なんとなーくは感じは掴めた
  - まぁ役には立たないけどやってよかった

### todo

- tokenize するときに`_ => (),`としてるがエラーにした方が良さそう
