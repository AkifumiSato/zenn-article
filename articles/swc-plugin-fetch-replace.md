---
title: "fetchをswc pluginで置き換える"
emoji: "🤪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["swc"]
published: false
---

- swcのfetch置き換えプラグインを作った
- 理由
  - 書いてみたかった
  - fetchを任意の関数で置き換えてMock強制とかしてみたかった
    - 実用性よか実験的
- swc docs
  - 結構わかりやすかった
  - 今回書かなかったがunitテストも書きやすそう
  - テストのmacroはドキュメントあってもいいかも
- 実装
  - playgroundみやすかった
  - 単なるfetchは置き換え簡単
  - global置き換えはちょっと面倒
  - 波動拳だけどASTいじるときはしょうがない気もする
- publish
  - あんまドキュメントに書いてない
  - wasiでいんだっけ？要調査
- 使ってみる
  - jestで使ってみた
    - 微妙
    - でもまぁとりあえず動いてる
- 今後
  - もうちょっと遊んでみるかも
