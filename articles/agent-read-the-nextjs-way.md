---
title: "AI Agentで『Next.jsの考え方』を活かす"
emoji: "📖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react"]
published: false
---

AI Agentの進化は目覚ましく、実装の品質も日々向上しています。しかし、開発者自身に明確な実装の完成イメージがある場合、それをAI Agentに出力させることは難しく、さまざまな工夫を必要とします。試行錯誤する手間暇を惜しんで、結局自分がドライバーとなって実装するこという経験をした方も多いのではないでしょうか。Agentに与えるコンテキストやプロンプトを最適化することで、より精度を高めることもできますが、ルールを整備してシンプルな命令から高い精度で実装できるように調整することがやはり理想です。しかし、自分にとって最適なルールを整備することもまた、大変な労力を伴います。

[『Next.jsの考え方』](https://zenn.dev/akfm/books/nextjs-basic-principle)は、筆者なりにNext.jsの設計やベストプラクティスをまとめたものです。筆者の場合、Next.jsにおける実装の完成イメージはこの本の内容に沿って生成されるため、「この本をAI Agentに読ませることで推論精度を高めることができるのではないか」と考えました。

本稿は、AI Agentに『Next.jsの考え方』を適用する筆者なりの工夫や、試行錯誤した内容について紹介します。

## 要約

『Next.jsの考え方』に沿って実装を進める場合、以下の条件下においてAI Agentの精度が向上します。

- AIのモデルは最新のコーディング特化なモデル（`claude-4-sonnet`^[執筆時時点でいくつかのモデルを比較した結果です。後継モデルが登場したらそちらの方がいいかもしれません。]など）
- リポジトリ内に、『Next.jsの考え方』に沿った参考コードを用意
- [Next.jsの考え方のマークダウン](https://github.com/AkifumiSato/zenn-article/tree/main/books/nextjs-basic-principle)をリポジトリ内に配置し、都度AI Agentに読ませるようルール設定

以下は検証に用いたリポジトリです。

https://github.com/AkifumiSato/akfm-nextjs-rules-demo/

## 前提

### AI Agent

- AI Agent: Claude Code, Cursor
- AI Model: `claude-4-sonnet`

### 『Next.jsの考え方』

『Next.jsの考え方』は文字通り、Next.jsの根底にある考え方に沿った設計やプラクティスについて、筆者なりに解説したものです。

https://zenn.dev/akfm/books/nextjs-basic-principle

## 解説

いくつかの前提条件とルールを整備することで、AI Agentに『Next.jsの考え方』を理解させ、実装品質の精度を高めることができます。

### 静的解析と単体テスト

『Next.jsの考え方』を活かすかどうかに関わらず、AI Agent系のツールを使用する場合、静的解析と単体テストは非常に重要です。AI Agentの実装が想定通りに振る舞っているかどうか、また既存の振る舞いに影響していないか、既存のコードとルールの整合制が取れているか、と言った観点をAI Agent自身が確認できることは、AI Agentの実装精度を高める上で非常に重要です。

特に、`claude-4-sonnet`は静的解析や単体テストのフィードバックをもとに改善のサイクルを回すことに長けています。

筆者が行った検証では、以下のツールを使用しました。

- Biome
- TypeScript
- Vitest

### 『Next.jsの考え方』に沿った参考実装

昨今のAI Agentは、リポジトリ内の既存実装を強く学習する傾向にあります。`CLAUDE.md`や`.cursor/rules/*`などのルールドキュメント内に参考実装を記載することも効果的ですが、筆者の体感ではリポジトリ内のコードの方が強く学習されるように感じます。

そもそも、ルールと既存実装で乖離してればAIにとっては混乱の元であるであろうことは想像できるので、ルールと整合性が取れた実装をリポジトリ内に用意しておくことは非常に重要です。

### 『Next.jsの考え方』のマークダウンコピー

AI Agentに『Next.jsの考え方』を読ませるためには、マークダウンファイルが最適だと考えられます。幸い、『Next.jsの考え方』はZennで無料公開しており、原著はパブリックリポジトリで管理されています。

https://github.com/AkifumiSato/zenn-article/tree/main/books/nextjs-basic-principle

https://github.com/AkifumiSato/zenn-article/blob/main/books/nextjs-basic-principle/part_1.md

`git clone`するなりGitHubからダウンロードするなり方法は様々ですが、これらのマークダウンファイルをコピーしてNext.jsのアプリケーションのリポジトリ内に配置します。これにより、AI Agentは任意のタイミングで『Next.jsの考え方』を参照できるようになります。

### ルールの作成

ルールはClaude Codeで`/init`で生成したものをベースに、『Next.jsの考え方』の格納場所や読むべきタイミングなどを設定します。以下は、筆者が実際に使用したルールの一例です。

:::message
『Next.jsの考え方』以外にも単体テストの記事など読み込むよう設定しています。
:::

```markdown:CLAUDE.md
# CLAUDE.md

このファイルは、このリポジトリでコードを扱う際のClaude Code (claude.ai/code) への指針を提供します。

...

## ドキュメント・ナレッジベース

`docs/akfm-knowledge/`ディレクトリには、React・Next.js・テストに関する包括的なベストプラクティスドキュメントが含まれています。

### 主要ドキュメント構成

#### 1. Next.js基本原理ガイド (`nextjs-basic-principle/`)

Next.js App Routerの包括的なガイド（36章構成）：

**Part 1: データ取得 (11章)**

- **参照タイミング**: データ取得パターンを実装する際
- **主要ファイル**:
  - `part_1_server_components.md` - Server Components設計の基本
  - `part_1_colocation.md` - データ取得の配置戦略
  - `part_1_request_memoization.md` - リクエスト最適化
  - `part_1_concurrent_fetch.md` - 並行データ取得
  - `part_1_data_loader.md` - DataLoaderパターン
  - `part_1_fine_grained_api_design.md` - API設計戦略
  - `part_1_interactive_fetch.md` - インタラクティブなデータ取得

**Part 2: コンポーネント設計 (5章)**

- **参照タイミング**: コンポーネント設計・リファクタリング時
- **主要ファイル**:
  - `part_2_client_components_usecase.md` - Client Components使用指針
  - `part_2_composition_pattern.md` - コンポジションパターン
  - `part_2_container_presentational_pattern.md` - Container/Presentational分離
  - `part_2_container_1st_design.md` - Container優先設計

**Part 3: キャッシュ戦略 (6章)**

- **参照タイミング**: パフォーマンス最適化・キャッシュ制御時
- **主要ファイル**:
  - `part_3_static_rendering_full_route_cache.md` - 静的レンダリング最適化
  - `part_3_dynamic_rendering_data_cache.md` - 動的レンダリング制御
  - `part_3_router_cache.md` - クライアントサイドキャッシュ
  - `part_3_data_mutation.md` - データ変更とキャッシュ無効化
  - `part_3_dynamicio.md` - 実験的キャッシュ改善

**Part 4: レンダリング戦略 (4章)**

- **参照タイミング**: レンダリング最適化・Streaming実装時
- **主要ファイル**:
  - `part_4_pure_server_components.md` - Server Component純粋性
  - `part_4_suspense_and_streaming.md` - プログレッシブローディング
  - `part_4_partial_pre_rendering.md` - 部分的事前レンダリング

**Part 5: その他の実践 (4章)**

- **参照タイミング**: 認証・エラーハンドリング実装時
- **主要ファイル**:
  - `part_5_request_ref.md` - リクエスト・レスポンス参照
  - `part_5_auth.md` - 認証・認可パターン
  - `part_5_error_handling.md` - エラーハンドリング戦略

#### 2. 単体記事 (`articles/`)

**フロントエンド単体テスト** (`articles/frontend-unit-testing.md`)

- **参照タイミング**: テスト戦略策定・テスト実装時
- **内容**:
  - Classical vs London school テスト手法
  - AAA（Arrange, Act, Assert）パターン
  - Storybookとの統合（`composeStories`）
  - テスト命名規則・共通セットアップパターン

### 参照ガイドライン

**参照タイミング**:

- 実装時には関連するドキュメントを必ず参照する
- ドキュメントを参照したら、「📖{ドキュメント名}を読み込みました」と出力すること

**機能実装時の参照優先順位**:

1. **データ取得実装** → Part 1のドキュメント群を参照
2. **コンポーネント設計** → Part 2のパターンを適用
3. **パフォーマンス最適化** → Part 3のキャッシュ戦略を活用
4. **レンダリング最適化** → Part 4のStreaming・PPR戦略を参照
5. **認証・エラーハンドリング** → Part 5の実践パターンを適用
6. **テスト実装** → `articles/frontend-unit-testing.md`を参照

**重要な設計原則**:

- **Server-First**: Server Componentsを優先し、必要時にClient Componentsを使用
- **データ取得の配置**: データを使用するコンポーネントの近くでデータ取得を実行
- **コンポジション**: 適切なコンポーネント分離とコンポジションパターンの活用
- **プログレッシブ強化**: JavaScript無効時でも機能する設計を心がける
```

冒頭省略したルールを含んだ全文は、以下より確認できます。

https://github.com/AkifumiSato/akfm-nextjs-rules-demo/blob/main/CLAUDE.md

## 定性評価

TBW

## 構成案

- 評価
  - 記事詳細ページを実装
  - 完璧ではないがそれなりに理解してるように見受けられた
  - 他リポジトリで試した時は、DataLoaderもうまく使った設計を行えていた
- 注意点
  - 本のみ
  - 参考コードのみ
  - 本のサマリーをルール設定
  - 既存実装のノイズ（未検証）
- 考察
  - ルール経由で本を読ませる方が精度は良かったように感じる
  - 多少具体例を含んでても本は抽象度が高いので、具体の参考コードがある方が強い
  - これらの要素どちらかが欠けて進めた場合、筆者の要求精度には到達しなかった
