---
title: "ポップコーンUIとSuspense"
emoji: "🍿"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ui", "frontend"]
published: false
---

## 導入

- ポップコーンUI：ポップコーンが弾けるように順次表示されるUI
- 避けるべきUX：個別フェッチによるスピナー乱立と連続的なガタつき
- AI Agent時代の開発：Reconciliation Loopと人間によるゴール定義の重要性
- フロントエンドの責務：UX要件の詳細定義とアンチパターンの認識が重要
- 本記事の目的：ポップコーンUI問題の提起と`<Suspense>`による解決策

## 前提

### Cumulative Layout Shift（CLS）

- Webパフォーマンスの潮流：ユーザー体験（UX）の定量化
- Core Web Vitals：UX定量化のための重要指標群
- CLS：予期せぬレイアウト変化（ガタつき）による不快感の計測

### Propsのバケツリレー（Props Drilling）

- 定義：上位でのデータフェッチと末端へのProps引き回し
- 歴史：React初期の忌避パターンで、Redux等により解決されていた
- 再燃：Next.js（Pages Router）の`getInitialProps`や`getServerSideProps`などにより、バケツリレー設計に回帰した

## 解説

### ポップコーンUI

- 定義：ポップコーンのように、複数のスピナーがランダムに置き換わっていくような体験
- 影響：Layout Shiftの発生=CLS悪化
- Tanstack Queryの例：`useQuery`によるローディング状態（`isLoading`）の取得をもとに、バケツリレーを避けて実装すると自然とポップコーンUIになる
- 構造的ジレンマ：hooksベースのAPIをもとに「自然な実装」をするとポップコーンUIになる

### Suspense

- 概要：React 18導入の非同期処理ハンドリング機能
- アプローチ：Hooksでの状態管理から、Suspend（Promise throw）と`<Suspense>`境界によるUXの抽象化へ
- TBW

## 私見

- 従来: Reactは`UI = f(state)`と表現していた時期がある
- パラダイムシフト：UIを「非同期的なもの」として捉える重要性
- Async React：`await UI = await f(await state)`
- Reactの進化：より良いUXの追求とその抽象化の歴史
