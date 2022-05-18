---
title: "ReactのStateのライフタイム"
emoji: "⌚️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: false
---

ReactはじめSPAのStateは大きく3種類、ローカルState・グローバルState・サーバーStateの3種類、これでおおよそのStateの分類が可能であると考えていました。これに対し会社の先輩から「参照スコープだけでなく、時間的グローバルなStateもあるよね」という意見をもらい、非常に納得しました。この時間軸スコープをRustからライフタイムという言葉を借りてStateの**ライフタイム**と呼称し、Stateの分類を自分なりに考察したいと思います。

## 参照スコープ軸の分類

まずこれまで自分が認識してた参照スコープにおけるStateの分類について触れておきます。以下の3種類で分類可能と考えていました。

- **Local State**: これは`useState`を利用することとほぼ同義です。コンポーネントがアンマウントされるまで生存する状態のことです。
- **Global State**: 複数のコンポーネントから参照されうるStateです。ReduxやRecoil、Context APIなどを通じて管理されることが多いかと思います。 
- **Server State**: クライアントサイドでキャッシュしてるAPIサーバーからのレスポンス値などです。SWRやReact Queryの内部で管理されてるものなどを指します。

細かい用語は違えど、以下の参考リンクではそれぞれ同じよう分類を行っています。

https://zenn.dev/yoshiko/articles/607ec0c9b0408d

以下のリンクでは上記と同様の分類か、データカテゴリによる分類（Data,Control/UI,Session,Communication,Locationの5つ）が可能とされています。

https://react-community-tools-practices-cheatsheet.netlify.app/state-management/overview/#types-of-state

Remixのコントリビュータのブログでは、Local/Globalなどの分類ではなく「全てのStateはUI Stateかサーバーキャッシュの2つに分けられる」と主張しています。

https://kentcdodds.com/blog/application-state-management-with-react#server-cache-vs-ui-state

これらの参考記事は非常に勉強になるので時間のある方はそれぞれ一読することをお勧めします。ここで主張したいこととしては、多くの場合Stateは参照スコープやデータカテゴリによってStateを分類しているということです。

## ライフタイムの定義

前述の分類に冒頭にあったように時間軸という観点を追加すると、これらの分類には**生存期間が異なるものが混在している**ことがわかります。例えばLocal Stateだけど変更のたびにsession storageに保存して`useEffect`で取得・復元することも可能です。session storageのkeyに復元可能な一意な値を指定することを考えれば、この例は実質的にGlobal Stateと見なすことができますが、先述の分類しか考えられてなかった時点では`useState`＝Local Stateと見なしてしまっていました。一方Global Stateでも同様に、一部の値をsession storageに保存するようなケースは十分考えられます。

このように先の分類のStateには、実際には生存期間が異なるものが混在しており、時間軸での実装方針について認識ずれる可能性があります。この生存期間がStateの**ライフタイム**というわけです。

### Rustにおけるライフタイム

ここで単語の借用元であるRustのライフタイムについても軽く触れておきます。興味のある方はRustのThe Bookと呼ばれる入門サイトがあるので、こちらを参照ください。

https://doc.rust-jp.rs/book-ja/ch10-03-lifetime-syntax.html

Rustのライフタイムは簡単にいうと、変数の生存期間を管理し生存期間を抜けた変数はメモリ的に解放されます。例えばJavascriptでは以下のコードが成り立ちます。

```js
let r = 0

{
  let x = 5
  r = x
}

console.log(r) // output: 5
```

一方Rustではほぼ似たようなコードを書くと、コンパイルエラーがおきます。

```rust
{
    let r;

    {
        let x = 5;
        r = &x;
    }

    println!("r: {}", r);
}
```

```
$ cargo run
   Compiling chapter10 v0.1.0 (file:///projects/chapter10)
error[E0597]: `x` does not live long enough
（エラー[E0597]: `x`の生存期間が短すぎます）
  --> src/main.rs:7:17
   |
7  |             r = &x;
   |                 ^^ borrowed value does not live long enough
   |                   (借用された値の生存期間が短すぎます)
8  |         }
   |         - `x` dropped here while still borrowed
   |          (`x`は借用されている間にここでドロップされました)
9  | 
10 |         println!("r: {}", r);
   |                           - borrow later used here
   |                            (その後、借用はここで使われています)

error: aborting due to previous error

For more information about this error, try `rustc --explain E0597`.
error: could not compile `chapter10`.

To learn more, run the command again with --verbose.
```

`&x`は変数`x`の参照を表しており、このエラーは変数`x`の生存期間が短すぎる旨を表しています。Rustでは変数がスコープを抜ける時にその変数のメモリを解放します。解放したメモリに対する参照が残ってしまうと、簡単にバグの温床となるでしょう。Rustではこれを防ぐために、Rustコンパイラは変数にライフタイムという注釈をつけて生存期間を管理しているわけです。先の例のライフタイムは以下のように、`r`には`'a`、`x`には`'b`というライフタイムが付与されて、`'b`の枠を超えて`x`は生存できない決まりになっています。

```rust
{
    let r;                // ---------+-- 'a
                          //          |
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
                          //          |
    println!("r: {}", r); //          |
}                         // ---------+
```

TBW

### ライフタイムを見落とすと起こる問題

## ライフタイムによるStateの分類

e.g. アコーディオンの開閉

### 1. Component mount

### 2. Javascript memory

### 3. Browser history

next.jsだと全部置き換えられてしまうので、使えなそう
https://github.com/vercel/next.js/blob/fe3d6b7aed5e39c19bd4a5fbbf1c9c890e239ea4/packages/next/shared/lib/router/router.ts#L1432

- Navigation API

### 4. Storage

- session
- local

### 5. Logical

- expireなど

## State Managementライブラリを分類

## まとめ

- 利用するライブラリの選定や、実装方針を考える時は時間軸によるStateの分類も考慮して考えよう。

Todo

- 図で主要ライブラリを分類
