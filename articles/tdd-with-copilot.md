---
title: "TDDでGitHub Copilotを使いこなす"
emoji: "🤖"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["githubcopilot", "tdd"]
published: false
---

GitHub Copilot、みなさん使ってますか？すでに多くの方が利用しており、「すでに必需品」という方から「提案の質に問題がある」「まだまだ使えない」という方まで、様々な意見を聞きます。新出のツールに対する感想は人によって習熟度が異なるので、一概にこれらの感想を受け入れるのではなく習熟度やコンテキストを見極めながら分析することが重要だと筆者は考えます。

さて、筆者はGitHub Copilotに対して非常にポイティブな立場です。GitHub Copilotは使い方次第で開発速度を格段に向上させることを身をもって体験しており、これからの時代においてはGitHub Copilot始め多くのAIツールを使いこなせるかどうかで、個人の開発速度に非常に大きな差が出ると考えています。重要なのは**使い方次第**と言う点です。静的解析同様、利用者側の手腕が大きく問われるツールであると筆者は感じています。

コマンドプロンプトエンジニアリングという言葉もあるように、AIツールを使いこなすには有用なヒントを与えることが重要です。ヒントが粗悪であれば、提案の品質も粗悪になってしまいます。また、GitHub Copilotはより小さい問題解決ほど得意としています。この点から筆者は、「TDDとGitHub Copilotは相性がいいのではないか」と考えて実践してみたところ、提案の品質を劇的に向上させることに成功しました。実際この数ヶ月実践し続けてみましたが、当初と感想は変わりません。

本稿はTDDとGitHub Copilotの相性について考察し、AI時代にこそ**TDDの習得が重要**であることを主張する記事です。

## TDDの定義と誤解

TDDとは**Test-Driven Development**（テスト駆動開発）の略称です。最近も考案者のKent Beck氏がTDDについて、本来の定義が弱まって伝わる「意味の希薄化」が発生しているとして改めて定義を説明し、話題になりました。以下はKent Beck氏の投稿の翻訳を含むt-wadaさんの記事です。

https://t-wada.hatenablog.jp/entry/canon-tdd-by-kent-beck

[テスト駆動開発の原著](https://www.ohmsha.co.jp/book/9784274217883/)もしくは上記記事を読んだことのない方はぜひ、というか絶対読んでください。以降本稿ではTDDの正しい定義と手順を知っている前提になります。

:::message
TDDについての誤解を助長したくはないため、ここでは概要の説明も意図的に省略しています。もっとみ短い説明は上記記事です。必ず上記リンクか原著を読んでください。
:::

## GitHub Copilotの学習

GitHub CopilotはGitHubが提供するAIツールです。いくつか機能がありますが、本稿で扱う最も重要な機能は**コード補完**です。入力途中のコードからGitHub Copilotは次のコードを予想・提案します。

https://docs.github.com/ja/copilot/about-github-copilot#github-copilot-%E3%81%AE%E6%A9%9F%E8%83%BD

この提案はGitHubに蓄えられてる膨大な学習データが利用されてるわけですが、他の学習要素として、利用者が開いているファイルタブからも強く学習します。提案内容は学習した結果に強く影響されるので、静的型定義や既存の実装・コメントはGitHub Copilotにとって大きなヒントになります。GitHub Copilotを使いこなす上では、**いかに良質なヒントを与えられるか**が利用者側のスキルとなるでしょう。

GitHub Copilotのコード補完は時に「AIとのペアプログラミング」と評されることがあります。現実のペアプログラミングでナビゲータの説明が上手だとドライバーは意図したコードをすぐに実装できるのと同じです。

### GitHub Copilotの苦手分野

前述の通り、GitHub Copilotの補完はAIとのペアプログラミングとも考えられます。現実のペアプログラミング同様に、ドライバー（GitHub Copilot）は誤った実装を提案をしてくる可能性もあります。そのため、ナビゲータ（利用者）は提案内容が正しいかどうか精査する**レビュー**を実施する必要があります。

補完された内容をただただ受け入れて実装が不完全だったというケースを時折聞きますが、そもそもGitHub CopilotはAIツール、つまり正しい内容を必ず提案するツールではなく、**ヒントに基づきそれらしい提案をするツール**です。AIツールを利用していようと自分が書いたコードとしてコミットする以上、そのコードに対する責任は利用者にあると筆者は考えています。この意識は非常に重要です。

## TDDとGitHub Copilot

序文にもある通り、筆者はTDDとGitHub Copilotは相性がいいと考えています。GitHub Copilotは具体的な問題点解決が得意なので、TDDによる小さく明確な課題解決を続けるスタイルはGitHub Copilotの得意な分野です。また、TODOリストやテストコードはプロダクションコードのヒントになるし、積み上げていくテストコードも大きなヒントです。これらのヒントにより、**GitHub Copilotの提案はTDDのサイクルごとにどんどん精度が高くなっていきます**。これは言葉だけでは伝わりづらい部分もあるので、実際に題材を元に実演してみたいと思います。

### 厳密なTDDのための退化

実演の前に、厳密なTDDを実践するために1つ補足説明をしたいと思います。GitHub Copilotは特性上、プロダクションコードの実装を一気に書き上げようとしてくることが多いです。そのため、TDDに従おうとするとあえて**実装を退化**させるフローが必要になることもあります。これにより次の提案をより質の高いものになる可能性もあるので、重要なフローだと筆者は考えています。

## GitHub CopilotとTDDを実際にやってみる

実際に2つの題材でGitHub CopilotとTDDを実践してみます。

### 技術選定

Copilotは言語やフレームワークに対して得意不得意があるので、TypeScriptで書くことにします。Nodeの.jsの方が得意かもしれませんが、筆者はほとんど差を感じたことがないので今回はDenoで実装してみます。

筆者は今回のようにライトに書くならDenoの方が好きと言うだけなので、深い意味はありません。

### 環境

筆者はWebStormを好んで利用しているので、WebStormで実演します。当然GitHub Copilotも有効になっています。VSCを使ってる方の方が多いでしょうが、今回の実演においてはほとんど差はありません。

### 実演(1) FizzBuzz

FizzBuzzを実装してみます。よく言われることですが出力などは検査したくないので、ここではFizzBuzz関数の実装のみを行います。

#### TODOリストを書く

TDDなのでまずはTODOリスト、と言いたいところですが、TODOリストもGitHub Copilotにとって重要なヒントなのでTODOリストはテストファイルに書きたいと思います。ファイル名は愚直に命名して問題ないだろうことから以下のファイルをまず作成します。

- `fizz_buzz.ts`
- `fizz_buzz_test.ts`

テストはファイルの最後にどんどん付け足していくでしょうから、`fib_buzz_test.ts`の末尾の方にTODOリストを書き始めます。途中まで書くと以下のようにTODOの記述自体も提案されることでしょう。

![fizz buzz todo init](/images/tdd-with-copilot/fizz-buzz/todo-of-fizz.png)

この補完を受け入れて改行すると次のTODOも提案されます。

![todo of buzz impl](/images/tdd-with-copilot/fizz-buzz/todo-of-buzz.png)

補完を受け入れつつ最初のTODOはこんなものでしょう。

![fizz buzz todo 1st](/images/tdd-with-copilot/fizz-buzz/todo-1.png)

#### テストを1つ書く

最初のTODOに対するテストを書きます。ここではテストケース名はそのまま利用していいでしょう。途中まで入力するとまたGitHub Copilotが提案してくれます。

![fizz buzz test case](/images/tdd-with-copilot/fizz-buzz/fizz-test-1.png)

筆者は**AAAパターン**でテストを書くことが多いので、ここではそれに倣ってテストを書いていきます。AAAについては以下の記事でも述べてる通りで、**Arrange**（準備）、**Act**（実行）、**Assert**（確認）の3つのフェーズに分けてテストを書くスタイルです。

https://zenn.dev/akfm/articles/frontend-unit-testing#aaa%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3

最初のテストなのでどういうAPI設計にするかもここで考える必要があります。もちろんプロダクションコードは空なのでそんな関数などないと警告されますが、まずは使い勝手から考えていきます。ここはシンプルに`fizzBuzz`関数とします。適宜補完を受け入れながら以下のようにしました。

![fizz buzz test impl](/images/tdd-with-copilot/fizz-buzz/fizz-test-2.png)

「`fizzBuzz`関数から定義してればTypeScriptの補完が得られたので損してるのでは？」と思うかもしれませんが、GitHub Copilotはこのエラーを元にプロダクションコードを補完してくれることも多いので、無駄にはなりません。

ではこのテストが失敗することを確認しましょう。以降も何度も実行するので`watch`しておきたいと思います。**ちゃんとテストが実行できていれば**失敗するはずです。

```shell-session
$ deno test --watch fizz_buzz_test.ts
Watcher Test started.
Check file:///Users/satouakifumi/work/git/sample/tdd-example-with-copilot/fizz_buzz_test.ts
error: TS2304 [ERROR]: Cannot find name 'fizzBuzz'.
  const result = fizzBuzz(3);
                 ~~~~~~~~
    at file:///Users/satouakifumi/work/git/sample/tdd-example-with-copilot/fizz_buzz_test.ts:15:18
```

ちゃんと失敗しました。前述のt-wadaさんの記事から引用すると

> より正確に表現するなら「リスト、レッド、グリーン、リファクタ」であるということです。

上記手順におけるリストとレッドが完了しました。

#### シンプルな実装を書く

次はテストが通るように、非常に質素な実装をします。`fizz_buzz.ts`に`export function`まで書いたら、以下のようになりました。

![fizz buzz impl](/images/tdd-with-copilot/fizz-buzz/fizz-buzz-impl-1.png)

これはFizzBuzzの実装完成系そのままですね。今回のように簡単な題材であればこれを受け入れるのも手でしょうが、それではTDDの実演とは言い難いのでここはあえて非常に質素な実装に退化させます。分岐を削って整えて、1行のみにしてしまいます。

![fizz buzz impl 2](/images/tdd-with-copilot/fizz-buzz/fizz-buzz-impl-2.png)

これでテストコード側でimportすればOKなはずです。

![fizz buzz test 3](/images/tdd-with-copilot/fizz-buzz/fizz-test-3.png)

しかしここでテスト結果を見ると失敗しています。

```shell-session
3の倍数の場合はFizzを返す => https://jsr.io/@std/testing/0.224.0/_test_suite.ts:191:10
error: AssertionError: Values are not equal.


    [Diff] Actual / Expected


-   fizz
+   Fizz
```

なんと、補完を受け入れてるうちに**大文字小文字の違いを見逃してしまいました**。TDDのプラクティスの1つに「2つのことを同時にするな」と言うのがあります。TDDだと最初にリストを書き出す時に実現したいことにちゃんと集中しているため、テストコード側で**この手のミスが起こりづらいのです**。このように、GitHub Copilotが完全でなくともTDDによって救われることが多々あるのが、筆者がGitHub CopilotとTDDが相性がいいと考える理由の1つです。

これはプロダクションコードの戻り値を`Fizz`にすればもちろんテストが通ってGREENになります。

```shell-session
3の倍数の場合はFizzを返す ... ok (0ms)

ok | 1 passed | 0 failed (1ms)
```

まだリファクタリングするほどプロダクションコードがないので、次のテストに進みます。

#### 2つ目のテスト

TBW: 5の倍数の実装から

## 構成

- 実演(1)
  - TODO Listを書く
  - テストを1つ書く
  - シンプルな実装を書く
  - リファクタを検討する
- 実演(2)
  - fetcherを実装し、httpステータスに応じて戻り値やthrowしてみよう
- 生産性の差
- まとめ

https://hackernoon.com/ja/github-copilot-%E3%83%AC%E3%83%93%E3%83%A5%E3%83%BC%E3%81%AF%E6%9C%AC%E5%BD%93%E3%81%AB%E9%96%8B%E7%99%BA%E9%80%9F%E5%BA%A6%E3%82%92-55%25-%E5%90%91%E4%B8%8A%E3%81%95%E3%81%9B%E3%81%BE%E3%81%99%E3%81%8B