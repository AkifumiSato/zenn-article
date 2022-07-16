---
title: "Next.jsはどうやってスクロール位置を復元するのか"
emoji: "📜"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

[Next.js](https://nextjs.org/)にはexperimental(実験的機能)で`scrollRestoration`というフラグが存在します。デフォルトでもSSGなど静的buildの場合にはスクロール位置を復元してくれますが、`getServerSideProps`利用時にはこのフラグを有効にしないとスクロール位置が復元されません。最近会社の先輩とこの辺りをいじってNext.jsに修正プルリクなど送っていたので、自分では気付けないような部分の知見も多く得られたので、備忘録兼ねてこのフラグが何を解決しようとして、どう実装されているのか解説したいと思います。

## Next.jsのスクロール復元挙動

### `getServerSideProps`なしのページの場合

- スクロール位置はブラウザによって復元される

### `getServerSideProps`ありのページの場合

- 非同期処理の場合、ブラウザ側での復元が先にきてしまう
- こう言った問題を scroll amnesiaと呼ぶ

### SPAではこう言った問題起きがち

- MPAだとブラウザがやってくれてるがSPAだと起きがち
- Vercelの社長の解説

### `experimental.scrollRestoration`

- このフラグによってgSSPもサポートできる
- 内部的にはgSSP完了後に頑張って位置を復元してる

## next.jsのscrollRestoration実装

### 12.2.0まで

- indexを内部にもってる
- 遷移時やpopstate時に更新してる

### 12.2.0以降

- 連番はよく壊れる
- PRのリンク

### 残ってる問題点

- リロード時に復元できない
- PRのリンク

## 余談:実はNext.jsで失ってるのはスクロールだけではない

### SPAでは遷移時に状態が破壊されがち

### vercelの社長とnext.jsの方針が噛み合ってない

- history壊してる
- 発火順おかしいとき

### recoil-sync-next
