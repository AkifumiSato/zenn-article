---
title: "Server Actions時代のformライブラリconform"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "conform", "nextjs"]
published: false
---

**conform**は[プログレッシブ・エンハンスメント](https://zenn.dev/cybozu_frontend/articles/think-about-pe)を意識して作られたReactのformライブラリです。

https://conform.guide/

[Remix](https://remix.run/)や[Next.js](https://nextjs.org/)などのフレームワークをサポートしています。特徴的なのがNext.jsの[Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)にも対応していることです。[react-hook-form](https://react-hook-form.com/)がServer Actions対応が[検証中](https://github.com/react-hook-form/react-hook-form/pull/11061)のため、他ライブラリを検討していた筆者には待望のライブラリでした。

本稿ではNext.js(App Router)におけるconformの使い方を中心に紹介します。

## Server Actions

TBW

### 基本的な使い方

### `useFormState`

### zod validation

TBW: 素のzod validation

## conformの特徴

公式ドキュメントによると、conformの特徴として以下が挙げられています。

- Progressive enhancement first APIs
- Type-safe field inference
- Fine-grained subscription
- Built-in accessibility helpers
- Automatic type coercion with Zod

## Next.js×conform

### インストール

### Server Actions

### カスタムエラー

### `getInputProps`/`getSelectProps`/`getTextareaProps`
