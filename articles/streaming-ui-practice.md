---
title: "Streaming UIプラクティス"
emoji: "⚡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["performance", "ui"]
published: true
---

アプリケーションユーザーにとってパフォーマンスとは、特定の指標に基づいて判断されるような定量的なものではなく、アプリケーションの総合的な体験を通じて感じる定性的なものです。そのため、ユーザーがアプリケーションを「遅い」と感じる要因は様々です。

Googleは以前より、[ユーザー中心のパフォーマンス](https://web.dev/articles/user-centric-performance-metrics?hl=ja)を提唱しています。[Core Web Vitals](https://developers.google.com/search/docs/appearance/core-web-vitals?hl=ja)はユーザーが感じるパフォーマンスを定量的に計測するために作成された測定指標で、現在最も重要なパフォーマンス指標の1つです。一方、Core Web Vitalsはパフォーマンスに関するUX全てを計測するものではありません。Core Web Vitalsを改善することは重要ですが、最終的にはUX全体を最適化する必要があります。

本稿では、ページ読み込み時に遅いと感じさせないために筆者がよく用いる、**Streaming UIのプラクティス**について解説します。

:::message alert
本稿はUX観点のプラクティスを解説するものです。技術的な実現方法については扱いません。
:::

## 要約

1. [早期レイアウト表示](#1-早期レイアウト表示)
2. [動的なUI要素の並行読み込み](#2-動的なUI要素の並行読み込み)
3. [ローダーやスケルトンは遅い時のみ](#3-ローダーやスケルトンは遅い時のみ)
4. [fade inアニメーション](#4-fade-inアニメーション)

## 前提

Streaming UIの採用について、以下の考え方を前提としています。

### SEOとJavaScript

Streaming UIの実現には、多くの場合JavaScriptによる処理が必要になります。コンテンツの表示にJavaScriptが必要というと、SEOへの影響を懸念される方もいるでしょう。

JavaScriptを必要とするコンテンツとSEOの関係については、以前は様々な影響があることが示唆されていましたが、2024年のVercelの調査によると、昨今では影響が限定的になっていることが判明しています。

詳細な内容は以下の記事をご参照ください。とても興味深い結果が記載されています。

https://vercel.com/blog/how-google-handles-javascript-throughout-the-indexing-process

## 解説

**Streaming UI**は、UIが徐々に読み込まれ表示されるようなUXのことを指し、ユーザーに「読み込みが遅い」と感じさせないために使われるテクニックです。

![Streaming UI](/images/streaming-ui-practice/streaming-ui.png)
_引用: [Next.jsの公式ドキュメント](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming)_

Streaming UIでユーザーの体験を改善するには、様々な配慮や工夫が必要です。

### 1. 早期レイアウト表示

アプリケーションのUXは、ユーザーの操作に対しできるだけ早くフィードバックを返すことが望ましいとされており、Streaming UIはユーザーへのフィードバックを段階的にするため、ページ遷移時に早期フィードバックが可能になります。

具体的には、レイアウトをできるだけ早く表示することで、ユーザーに対する無応答時間を減らすことができます。

#### 例

以下は、早期にレイアウトを表示し、遅れて記事一覧が表示される様子です。

![レイアウト表示](/images/streaming-ui-practice/layout.png)

レイアウトを構成するUI要素は、静的もしくはキャッシュ可能データを中心に構成することで、より早期に表示することが可能になります。

#### 注意点

ページ読み込み中のレイアウト変更は、ユーザーに不快感を与えます。これはCore Web Vitalsにおける[Cumulative Layout Shift（CLS）](https://web.dev/articles/cls?hl=ja)という指標で計測することができるので、[Light House](https://developer.chrome.com/docs/lighthouse/overview?hl=ja)などを利用してCLSを最小化することを心がけましょう。

### 2. 動的なUI要素の並行読み込み

Streaming UIに限らず、データ取得の並行性は重要です。

![Sequential Loading](/images/streaming-ui-practice/sequential-loading.png)

![Concurrent Loading](/images/streaming-ui-practice/concurrent-loading.png)
_引用: [Next.jsの公式ドキュメント](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming#what-is-streaming)_

Streaming UIにおいては、静的なUI要素をレイアウトに含めつつ、動的なUI要素は並行な読み込みになるよう設計しましょう。

#### 例

以下は、記事一覧とログイン状態を表すユーザーのアバターが並行して読み込まれ、徐々に表示される様子です。アバターが少し遅れて表示されています。

![動的なUI要素の並行読み込み](/images/streaming-ui-practice/concurrent-load.png)

#### 注意点

UX上並行の読み込んでるように見えるだけでなく、実際に**データ取得が並行になるよう実装する必要がある**ことに注意しましょう。

### 3. ローダーやスケルトンは遅い時のみ

読み込みが遅い時には、ローダーやスケルトンを使ってユーザーに対し「読み込みに時間がかかっています」ということを伝えましょう。

#### 例

以下は、記事の読み込みに時間がかかっている際に、スケルトンを表示する様子です。

![ローダーやスケルトンは遅い時のみ](/images/streaming-ui-practice/skeleton.png)

記事カセットの高さが自明な場合、スケルトンでも同様の高さにすることで、レイアウトシフトを防ぐことができます。

#### 注意点

ローダーやスケルトンは読み込みが速い時に表示すると、画面がチラついて見えてしまいます。そのため、筆者は読み込み速度に応じて以下のような判断軸でこれらを使用しています。

| UI要素の表示速度           | ローダー・スケルトンの要否 |
| :------------------------- | :------------------------- |
| **安定して速い**           | 不要                       |
| **安定して遅い**           | 必要                       |
| **不安定（遅い時がある）** | 遅い時だけ出す             |

速い/遅いの判断は主観ですが、表示に1秒近くかかるあたりから筆者は「遅い」と判断しています。

なお、ローダーやスケルトンを「遅い時だけ出す」には様々な実装方法が考えられますが、筆者はよくCSSアニメーションの[animation-delay](https://developer.mozilla.org/en-US/docs/Web/CSS/animation-delay)を利用して遅い時だけスケルトンをfade inするようにしています。

### 4. fade inアニメーション

遅れて表示されるコンテンツには、fade inアニメーションを適用しましょう。fade inアニメーションは、Streaming UIで発生するちらつきを抑え、ユーザーの違和感を軽減します。

#### 例

以下は記事一覧が読み込まれる時に、fade inアニメーションする様子です。この例では記事一覧の読み込みが安定して高速であるため、スケルトンは表示されません。

![fade inアニメーション](/images/streaming-ui-practice/fade-in-animation.gif =300x)

アニメーションgifなので少々わかりづらいですが、アニメーションがあることによって、レイアウトと記事一覧が同時に読み込まれたかのような体験になります。

## まとめ

パフォーマンスはユーザーにとって当たり前の品質であり、損なえばユーザーに不快感を与えます。そしてパフォーマンスチューニングは、開発者たちが当事者意識を持って改善に臨まなければ、なかなか改善しません。

Core Web Vitalsなど定量化された指標でパフォーマンスを計測し、改善することは非常に重要です。同様に、UXの工夫で「アプリケーションが遅い」と感じさせないことも重要です。

本稿が、パフォーマンス改善の観点を広く持つきっかけになれば幸いです。
