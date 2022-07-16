---
title: "Next.jsはどうやってスクロール位置を復元するのか"
emoji: "📜"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

[Next.js](https://nextjs.org/)にはexperimental(実験的機能)で`scrollRestoration`というフラグが存在します。最近会社の先輩とこの辺りをいじってて、自分では気付けないような部分の知見も多く得られたので、備忘録兼ねてこのフラグが何を解決しようとして、どう実装されているのか解説したいと思います。

- scrollRestorationとは
  - SPAとscroll amnesia
    - MPAだとブラウザがやってくれてる
  - SPAではこう言った問題起きがち
    - Vercelの社長の解説
- next.jsのrestore実装
  - 12.2.0まで
    - indexを内部にもってる
    - 遷移時やpopstate時に更新してる
  - 12.2.0以降
    - 連番はよく壊れる
    - PRのリンク
  - 残ってる問題点
    - リロード時に復元できない
    - PRのリンク
- scroll restorationから見えてくるSPAの問題点
  - ちょっと余談
  - SPAでは遷移時に状態が破壊されがち
  - vercelの社長とnext.jsの方針が噛み合ってない
      - history壊してる
      - 発火順おかしいとき
  - recoil-sync-next
