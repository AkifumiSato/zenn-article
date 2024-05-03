---
title: "TDDでGitHub Copilotを使いこなす"
emoji: "🤖"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["githubcopilot", "tdd"]
published: false
---

GitHub Copilot、みなさん使ってますか？すでに多くの方が利用しており、「すでに必需品」という方から「提案の質に問題がある」「まだまだ使えない」という方まで、様々な意見を聞きます。筆者はGitHub Copilotに対して非常にポイティブな立場です。GitHub Copilotは使い方次第で開発速度を格段に向上させることを身をもって体験しており、これからの時代においてはGitHub CopilotなどのAIツールを使いこなせるかどうかで、個人の開発速度に非常に大きな差が出ると考えています。

重要なのは**使い方次第**と言う点です。静的解析同様、利用者側の手腕が大きく問われるツールであると筆者は感じています。コマンドプロンプトエンジニアリングという言葉もあるように、AIツールを使いこなすには有用なヒントを与えることが重要です。ヒントが粗悪であれば、提案の品質も粗悪になってしまいます。良質なヒントと、明確で小さな問題を与えればGitHub Copilotは良質な提案を提供できます。この点から筆者は、「TDDとGitHub Copilotは相性がいいのではないか」と考えて実践してみたところ、GitHub Copilotの提案の品質を劇的に向上させることに成功しました。実際この数ヶ月実践し続けてみましたが、当初と感想は変わりません。

本稿はTDDとGitHub Copilotの相性について考察し、AI時代にこそ**TDDの習得が重要**であることを主張する記事です。

## TDDの定義と誤解

TDDとは**Test-Driven Development**（テスト駆動開発）の略称です。最近も考案者のKent Beck氏がTDDについて、本来の定義が弱まって伝わる「意味の希薄化」が発生しているとして改めて定義を説明し、話題になりました。以下はKent Beck氏の投稿の翻訳を含むt-wadaさんの記事です。

https://t-wada.hatenablog.jp/entry/canon-tdd-by-kent-beck

[テスト駆動開発の原著](https://www.ohmsha.co.jp/book/9784274217883/)もしくは上記記事を読んだことのない方はぜひ、というか絶対読んでください。以降本稿ではTDDの正しい定義と手順を知っている前提になります。

:::message
TDDについての誤解を助長したくはないため、ここでは概要の説明も意図的に省略しています。最も短く正確な説明は上記の記事です。必ず上記リンクか原著を読んでください。
:::

## GitHub Copilotの学習

GitHub CopilotはGitHubが提供するAIツールです。いくつか機能がありますが、本稿で扱う最も重要な機能は**コード補完**です。入力途中のコードからGitHub Copilotは次のコードを予想・提案します。

https://docs.github.com/ja/copilot/about-github-copilot#github-copilot-%E3%81%AE%E6%A9%9F%E8%83%BD

この提案はGitHubに蓄えられてる膨大な学習データが利用されてるわけですが、他の学習要素として、利用者が開いているファイルタブからも強く学習します。提案内容は学習した結果に強く影響されるので、静的型定義や既存の実装・コメントはGitHub Copilotにとって大きなヒントになります。GitHub Copilotを使いこなす上では、**いかに良質なヒントを与えられるか**が利用者側のスキルとなるでしょう。

GitHub Copilotのコード補完は時に「AIとのペアプログラミング」と評されることがあります。現実のペアプログラミングでナビゲータの説明が上手だとドライバーは意図したコードをすぐに実装できるのと同じです。

### GitHub Copilotの苦手分野

前述の通り、GitHub Copilotの補完はAIとのペアプログラミングとも考えられます。現実のペアプログラミング同様に、ドライバー（GitHub Copilot）は誤った実装を提案をしてくる可能性もあります。そのため、ナビゲータ（利用者）は提案内容が正しいかどうか精査する**レビュースキル**が求められます。

補完された内容をただただ受け入れて実装が不完全だったというケースを時折聞きますが、そもそもGitHub CopilotはAIツール、つまり正しい内容を必ず提案するツールではなく**ヒントに基づきそれらしい提案をするツール**です。AIツールを利用していようと自分が書いたコードとしてコミットする以上、そのコードに対する責任は利用者にあると筆者は考えています。この意識は非常に重要です。

## TDDとGitHub Copilot

さて、序文にもある通り筆者はTDDとGitHub Copilotは相性がいいと考えています。GitHub Copilotは具体的な問題点解決が得意なので、TDDによる小さく明確な課題解決を続けるスタイルはGitHub Copilotの得意な分野です。また、TODOリストやテストコードはプロダクションコードのヒントになるし、積み上げていくテストコードも大きなヒントです。これらのヒントにより、GitHub Copilotの提案は**TDDのサイクルごとにどんどん精度が高くなっていきます**。これは言葉だけでは伝わりづらい部分もあるので、実際に題材を元に実践してみたいと思います。

### 厳密なTDDのための退化

実践の前に、厳密なTDDを実践するために1つ補足説明をしたいと思います。GitHub Copilotは特性上、プロダクションコードの実装を一気に書き上げようとしてくることが多いです。そのため、TDDに従おうとするとあえて**実装を退化**させるフローが必要になることもあります。一気に補完される実装案が正しいとは限りませんし、TDDは設計手法の一面も持っているので実装を退化させてでもTDDに則ったフローで実装することが重要だと筆者は考えています。

:::message
[Clean Craftsmanship](https://www.kadokawa.co.jp/product/302206001224/)という本でもTDDにおける実装の「行き詰まり」の解消法として、「実装を退化させる」というプラクティスが紹介されています。馬鹿馬鹿しいように感じられるかもしれませんが、これもれっきとしたTDDのプラクティスの1つです。
:::

## GitHub CopilotとTDDを実際にやってみる

実際にTDDをGitHub Copilotと実践してみます。ここではお決まりのFizzBuzzを実装してみましょう。

### 技術選定

Copilotは言語やフレームワークに対して得意不得意があるので、TypeScriptで書くことにします。Nodeの.jsの方が得意かもしれませんが、筆者はほとんど差を感じたことがないので今回はDenoで実装してみます。

筆者は今回のようにライトに書くならDenoの方が好きと言うだけなので、深い意味はありません。

### 環境

筆者はWebStormを好んで利用しているので、WebStormで実践します。当然GitHub Copilotも有効になっています。VSCを使ってる方の方が多いでしょうが、今回の実践においてはほとんど差はありません。

### 要件

「どこまでテストするか」などの議論でもよく言われることですが、出力はテストしづらいがテストしてもあまり旨みがないと言われています。なのでここではFizzBuzzの文字列を返す関数の実装のみを行います。

### 最初のサイクル

TDDなのでまずはTODOリスト、と言いたいところですが、TODOリストもGitHub Copilotにとって重要なヒントなのでTODOリストはテストファイルに書きたいと思います。ファイル名は愚直に命名して問題ないだろうことから以下のファイルをまず作成します。

- `fizz_buzz.ts`
- `fizz_buzz_test.ts`

テストはファイルの最後にどんどん付け足していくでしょうから、`fib_buzz_test.ts`の末尾の方にTODOリストを書き始めます。途中まで書くと以下のようにTODOの記述自体も提案されることでしょう。

![fizz buzz todo init](/images/tdd-with-copilot/fizz-buzz/todo-of-fizz.png)

:::message
少しわかりづらいかもしれませんが、WebStormだと上記のようにコメントアウトが濃い緑色、GitHub Copilotの提案が灰色で表示されます。
:::

この補完を受け入れて改行すると次のTODOも提案されます。

![todo of buzz impl](/images/tdd-with-copilot/fizz-buzz/todo-of-buzz.png)

補完を受け入れつつ最初のTODOはこんなものでしょう。

![fizz buzz todo 1st](/images/tdd-with-copilot/fizz-buzz/todo-1.png)

#### テストを1つ書く

最初のTODOに対するテストを書きます。ここではテストケース名はそのまま利用していいでしょう。途中まで入力するとまたGitHub Copilotが提案してくれます。

![fizz buzz test case](/images/tdd-with-copilot/fizz-buzz/fizz-test-1.png)

筆者は**AAAパターン**でテストを書くことが多いので、ここではそれに倣ってテストを書いていきます。AAAについては以下の記事でも述べてる通りで、**Arrange**（準備）、**Act**（実行）、**Assert**（確認）の3つのフェーズに分けてテストを書くスタイルです。

https://zenn.dev/akfm/articles/frontend-unit-testing#aaa%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3

最初のテストなのでどういうAPI設計にするかもここで考える必要があります。もちろんプロダクションコードは空なのでそんな関数などないと警告されますが、まずは使い勝手から考えていきます。ここはシンプルに`fizzBuzz`関数と想定しましょう。

![fizz buzz test impl](/images/tdd-with-copilot/fizz-buzz/fizz-test-2.png)

GitHub Copilotからもテストコード案は提案されますが、最初のテストコードではそれをそのまま受け入れるのではなく適宜修正するか自身でしっかり書きましょう。ここで書いたコードが後続の作業で補完されるサンプルデータにもなるので、手本として適当に書いたりせず変数名1つとってもしっかり考えて命名することをお勧めします。

またここで、「`fizzBuzz`関数から定義してればTypeScriptの補完が得られたので損してるのでは？」と思うかもしれませんが、GitHub Copilotがこのエラーを元にプロダクションコードを補完してくれることも多いので、無駄にはなりません。

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

ちゃんと失敗しました。TDDにおける「リスト」と「レッド」が完了しました。

#### シンプルな実装を書く

次はテストが通るように、非常に質素な実装をします。`fizz_buzz.ts`に`export function`まで書いたら、以下のようになりました。

![fizz buzz impl](/images/tdd-with-copilot/fizz-buzz/fizz-buzz-impl-1.png)

これはFizzBuzzの実装完成系そのままですね。今回のように簡単な題材であればこれを受け入れるのも手でしょうが、それではTDDの実践とは言い難いのでここはあえて非常に質素な実装に退化させます。分岐を削って整えて、1行のみにしてしまいます。

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

まだリファクタリングするほどプロダクションコードがないので、TODOリストにチェックを入れて次のテストに進みます。

### 2週目

次は「5の倍数の場合はBuzzを返す」のテストコードです。

![buzz test 1](/images/tdd-with-copilot/fizz-buzz/buzz-test-1.png)

この段階ですでに、**GitHub Copilotが良質な提案をするようになってきた**ことがみて取れます。筆者の経験上、TDDに限らず多くの場合2つ目のテストから提案の質が一気に高くなります。まさしくAIとのペアプログラミングをしてる感覚です。

```shell-section
5の倍数の場合はBuzzを返す => https://jsr.io/@std/testing/0.224.0/_test_suite.ts:191:10
error: AssertionError: Values are not equal.


    [Diff] Actual / Expected


-   Fizz
+   Buzz
```

テストが失敗したので、プロダクションコードを修正します。`return "Fizz";`を消して`_n`を`n`にします。

![buzz impl 1](/images/tdd-with-copilot/fizz-buzz/buzz-impl-1.png)

まさに今欲しいコードが提案されました。今回は題材が簡単なのでばかばかしく感じるかもしれませんが、TDDのフローで考えるとお手本のようなシンプルな実装です。しっかりテストも通ってます。

```shell-session
3の倍数の場合はFizzを返す ... ok (0ms)
5の倍数の場合はBuzzを返す ... ok (0ms)

ok | 2 passed | 0 failed (1ms)
```

このようにプロダクションコードの一部を修正する時にも、GitHub Copilotが意図を組んで提案してくれるようになります。最初から全部を実装するような補完を提案してきた時と違い、GitHub CopilotもしっかりTDDに寄り添ってくれています。

まだリファクタリングするほどでもないので、次のテストに進みます。

### 3週目

3つ目のテストは「3と5の倍数の場合はFizzBuzzを返す」です。

![fizzbuzz test 1](/images/tdd-with-copilot/fizz-buzz/fizz-buzz-test-1.png)

ここでもGitHub Copilotがいい感じにちょうど実装したいテストコードを補完してくれています。このテストも当然失敗するので、プロダクションコードを修正します。`fizzBuzz`関数の先頭に`FizzBuzz`を返す分岐を追加します。

![fizzbuzz impl 3](/images/tdd-with-copilot/fizz-buzz/fizz-buzz-impl-3.png)

この提案内容を受け入れると無事テストが通ります。

```shell-session
3の倍数の場合はFizzを返す ... ok (0ms)
5の倍数の場合はBuzzを返す ... ok (0ms)
3と5の倍数の場合はFizzBuzzを返す ... ok (0ms)

ok | 3 passed | 0 failed (1ms)
```

#### リファクタリング

ここで初めてリファクタリングを行います。先頭の分岐は「`15`で割り切れる場合」に変更しても問題ないはずです。ついでに`return`しかないブロックも1行にまとめてしまいます。このようなリファクタも途中でGitHub Copilotが意図を汲み取って補完してくれるシーンがみられます。

![fizzbuzz refactor 1](/images/tdd-with-copilot/fizz-buzz/refactor-1.png)

修正後もテストは引き続きGREENです。

### 4週目

最後に「それ以外の場合はそのままの数値を返す」のテストを書きます。

![other test](/images/tdd-with-copilot/fizz-buzz/other-test-1.png)

ここに来てついに、**テストケース名まで含め書きたいコードが全て補完**されるようになりました。このように学習を重ねることで、GitHub Copilotの提案の質はどんどん向上していきます。

これももちろん失敗するので、プロダクションコードを修正します。

![default impl 1](/images/tdd-with-copilot/fizz-buzz/default-impl-1.png)
![default impl 2](/images/tdd-with-copilot/fizz-buzz/default-impl-2.png)

プロダクションコードの補完もしっかり筆者の頭にあるコードを完璧に補完してくれました。

```shell-session
3の倍数の場合はFizzを返す ... ok (0ms)
5の倍数の場合はBuzzを返す ... ok (0ms)
3と5の倍数の場合はFizzBuzzを返す ... ok (0ms)
それ以外の場合はそのままの数値を返す ... ok (0ms)

ok | 4 passed | 0 failed (1ms)
```

これでFizzBuzzの実装は完了です。リファクタリングも特に必要ないでしょうが、追加でもう少しテストを整理してみましょう。

### 5週目(テストケースの整理)

TBW: 各倍数のテストケースを追加

## まとめ

TBW
