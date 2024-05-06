---
title: "AI時代にこそTDDだと思う話"
emoji: "🤖"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["githubcopilot", "tdd"]
published: true
---

GitHub Copilot、みなさん使ってますか？すでに多くの方が利用しており、「ないと困る」という方から「提案の質に問題がある」「まだまだ使えない」という方まで、様々な意見を聞きます。

筆者はGitHub Copilotに対して非常にポイティブな立場です。GitHub Copilotは使い方次第で開発速度を格段に向上させることを身をもって体験しており、これからの時代においてはGitHub CopilotなどのAIツールを使いこなせるかどうかで、個人の開発速度に非常に大きな差が出ると考えています。

重要なのは**使い方次第**と言う点です。前述のように様々な感想が溢れているのはAIツールの習熟度が大きく影響しているようにも感じます。AIツールは静的解析同様、利用者側の手腕が大きく問われるツールであると筆者は感じています。コマンドプロンプトエンジニアリングという言葉もあるように、AIツールを使いこなすには**良質なヒント**を与えることが重要です。ヒントが粗悪であれば、提案の品質も粗悪になってしまいます。良質なヒントを与えればGitHub Copilotは良質な提案を提供してきます。この点から筆者は「TDDとGitHub Copilotは相性がいいのではないか」と考えて実践してみたところ、GitHub Copilotの提案の品質が劇的に向上しました。実際筆者はこの数ヶ月、GitHub CopilotとのTDDを実践し続けてみましたが、当初と感想は変わりません。

本稿はTDDとGitHub Copilotの相性について考察・実践し、AI時代にこそ**TDDの習得が重要である**ことを主張する記事です。

## TDDの定義と誤解

**TDD**とはTest-Driven Development（テスト駆動開発）の略称です。最近も考案者のKent Beck氏がTDDについて、本来の定義が弱まって伝わる「意味の希薄化」が発生しているとして改めて定義を説明し、話題になりました。以下はKent Beck氏の投稿の翻訳を含むt-wadaさんの記事です。

https://t-wada.hatenablog.jp/entry/canon-tdd-by-kent-beck

[テスト駆動開発の原著](https://www.ohmsha.co.jp/book/9784274217883/)もしくは上記記事を読んだことのない方はぜひ、というか絶対読んでください。動画で見たいと言う方は、数年前の[TDD Boot Camp](https://www.youtube.com/watch?v=Q-FJ3XmFlT8)も参考になるでしょう。

以降本稿ではTDDの正しい定義と手順を知っている前提になります。

:::message alert
安易な説明でTDDの誤解を助長するようなことはしたくないため、ここでは**概要の説明も意図的に省略**しています。最も短く正確な説明は上記の記事です。TDDについては多くの誤解が蔓延しているので、原著を読んだことがない方は必ず上記リンクの記事を読んでください。
:::

## GitHub Copilotの学習

**GitHub Copilot**はGitHubが提供するAIツールです。いくつか機能がありますが、本稿で扱う最も重要な機能は**コード補完**です。入力途中のコードからGitHub Copilotは次のコードを予想・提案します。

https://docs.github.com/ja/copilot/about-github-copilot#github-copilot-%E3%81%AE%E6%A9%9F%E8%83%BD

この提案はGitHubに蓄えられてる膨大な学習データが利用されてるわけですが、他の学習要素として、利用者のIDE上で隣接するファイルタブからも強く学習します。

https://qiita.com/ryosuke-horie/items/39ba28294fc4bbfc8722#%E9%96%A2%E9%80%A3%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%92%E9%9A%A3%E6%8E%A5%E3%82%BF%E3%83%96%E3%81%AB%E9%96%8B%E3%81%84%E3%81%A6%E3%81%8A%E3%81%8F

当然のことながら提案内容は学習内容に強く影響されるので、静的型定義や既存の実装・コメント・命名はGitHub Copilotにとって大きなヒントになります。GitHub Copilotを使いこなす上では、**いかに良質なヒントを与えられるか**が利用者側のスキルとなるでしょう。

### AIとのペアプログラミング

余談ですが、GitHub Copilotとの開発体験は時に「AIとのペアプログラミング」と評されることがあります。現実のペアプログラミング同様、ナビゲータ（利用者）の説明が上手だとドライバー（GitHub Copilot）は意図したコードをすぐに提案できます。

### GitHub Copilotの苦手分野

現実のペアプログラミング同様、ドライバーであるGitHub Copilotは誤った実装を提案をしてくる可能性もあります。そのため、利用者は**提案内容が正しいかどうかレビュー**する必要があります。

補完された内容をただただ受け入れて実装が不完全だったというケースを時折聞きますが、そもそもGitHub CopilotはAIツール、つまり正しい内容を必ず提案するツールではなく**ヒントに基づきそれらしい提案をするツール**です。AIツールを利用していようと自分が書いたコードとしてコミットする以上、そのコードに対する責任は利用者にあると筆者は考えています。この意識は非常に重要です。

## TDDとGitHub Copilot

さて、序文にもある通り筆者はTDDとGitHub Copilotは相性がいいと考えています。AIツールは具体的な問題点解決が得意なので、TDDが生み出す小さく明確な問題はGitHub Copilotの得意分野です。また、TODOリストやテストコードはプロダクションコードや次のテストコードのヒントになります。これらのヒントにより、GitHub Copilotの提案は**TDDのサイクルごとにどんどん精度が高くなっていきます**。まさしくペアプログラミング体験です。これは言葉だけでは伝わりづらい部分もあるので、本稿の残りパートではこれを実践する様子を紹介したいと思います。

### 厳密なTDDのための退化

実践に入る前に、GitHub CopilotとのTDDについて1つ補足説明をしたいと思います。GitHub Copilotは特性上、プロダクションコードの実装を一気に書き上げようとしてくることが多いです。そのため、TDDに従おうとすると**あえて実装を退化させる**フローが必要になることもあります。もちろんケースバイケースなので一概には言えませんが、一気に補完される実装案が正しいとは限りませんし、TDDは設計手法の一面も持っているので実装を退化させてでもTDDに則ったフローで実装することが重要だと筆者は考えています。

:::message
[Clean Craftsmanship](https://www.kadokawa.co.jp/product/302206001224/)という本でもTDDにおける「実装の行き詰まり」の解消法として、「実装を退化させる」というプラクティスが紹介されています。馬鹿馬鹿しいように感じられるかもしれませんが、小さな問題に分解して1つづつ丁寧に解消することで得られる設計の見直しはTDDにおいて非常に重要な要素です。
:::

## TDDをGitHub Copilotと実践する

実際にTDDをGitHub Copilotと実践してみます。ここではお決まりのFizzBuzzを実装してみましょう。

### 技術選定

GitHub Copilotは言語やフレームワークに対して得意不得意もあるので、今回はGitHub Copilotが得意とする言語でもあるTypeScriptで書くことにします。ランタイムについてはNodeの.jsの方が得意かもしれませんが、少なくとも筆者は大きな差を感じたことがないので今回はDenoで実装してみます。

筆者は今回のようにライトに書くならDenoの方が好きと言うだけなので、深い意味はありません。（[Denoはいいぞ...](https://zenn.dev/mizchi/articles/deno-first-choice)）

### IDE

筆者はWebStormを好んで利用しているので、WebStormで実践します。当然GitHub Copilotも有効になっています。VSCを使ってる方の方が多いでしょうが、今回の実践においてはほとんど差はありません。

### 実装要件

「どこまでテストするか」などの議論でもよく言われることですが、出力はテストしづらいがテストしてもあまり旨みがないと言われています。なのでここではFizzBuzzの文字列を返す関数の実装のみを行い、出力や連続した数列の生成は範囲外とします。

### 最初のサイクル: Fizz

TDDなのでまずはTODOリストと言いたいところですが、TODOリストもGitHub Copilotにとって重要なヒントなのでTODOリストはテストファイルに書きたいと思います。なのでまずはファイル作成から始めます。実装要件的にも愚直に命名して問題ないだろうことからファイル名は以下とします。

- `fizz_buzz.ts`
- `fizz_buzz_test.ts`

:::message
Denoに不慣れな方のために補足しておくと、Denoのファイル命名は[スネークケースが推奨](https://docs.deno.com/runtime/manual/references/contributing/style_guide#use-underscores-not-dashes-in-filenames)されています。
:::

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

最初のTODOに対するテストを書きます。ここではテストケース名はTODOリストのものをそのまま利用していいでしょう。途中まで入力するとまたGitHub Copilotがいい具合に提案してくれます。

![fizz buzz test case](/images/tdd-with-copilot/fizz-buzz/fizz-test-1.png)

筆者は**AAAパターン**でテストを書くことが多いので、ここではそれに倣ってテストを書いていきます。AAAについては以下の記事でも述べてる通りで、**Arrange**（準備）、**Act**（実行）、**Assert**（確認）の3つのフェーズに分けてテストを書くスタイルです。ActとAssertは必ず必要ですがArrangeは必要な時のみ記載するものなので、今回のサンプル実装だとどのケースでも不要で省略されています。

本稿の主題ではないので、詳しく知りたい方は以下の記事をご参照ください。

https://zenn.dev/akfm/articles/frontend-unit-testing#aaa%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3

最初のテストなのでどういうAPI設計にするかもここで考える必要があります。もちろんプロダクションコードは空なのでそんな関数などないと警告されますが、まずは使い勝手から考えていきます。ここはシンプルに`fizzBuzz`関数と想定しましょう。

![fizz buzz test impl](/images/tdd-with-copilot/fizz-buzz/fizz-test-2.png)

GitHub Copilotからもテストコード案は提案されますが、最初のテストコードではそれをそのまま受け入れるのではなく適宜修正するか自身でしっかり書きましょう。**ここで書いたコードが後続の作業で補完されるサンプルデータにもなる**ので、手本として適当に書いたりせず変数名1つとってもしっかり考えて命名することをお勧めします。

またここで、「`fizzBuzz`関数から定義してればTypeScriptの補完が得られたので損してるのでは？」と思うかもしれませんが、GitHub Copilotがこのエラーを元にプロダクションコードを補完してくれることも多いので、無駄にはなりません。

ではこのテストが失敗することを確認しましょう。以降も何度も実行するので`watch`しておきたいと思います。ちゃんとテストが実行できていれば以下のように関数が見つからずに失敗するはずです。

```shell-session
$ deno test --watch fizz_buzz_test.ts
Watcher Test started.
Check file:///Users/xxxx/tdd-example-with-copilot/fizz_buzz_test.ts
error: TS2304 [ERROR]: Cannot find name 'fizzBuzz'.
  const result = fizzBuzz(3);
                 ~~~~~~~~
    at file:///Users/xxxx/tdd-example-with-copilot/fizz_buzz_test.ts:15:18
```

これで、TDDにおける「LIST」と「RED」が完了しました。

#### シンプルな実装を書く

次はテストが通るように、非常に質素な実装をします。`fizz_buzz.ts`に`export function`まで書いたら、以下のようになりました。

![fizz buzz impl](/images/tdd-with-copilot/fizz-buzz/fizz-buzz-impl-1.png)

これはFizzBuzzの実装完成系そのままですね。今回のように簡単な題材であればこれを受け入れるのも手でしょうが、今回はTDDの実践のためここではあえて非常に質素な実装に退化させます。今回に限らず、提案される実装が少々多い・複雑だななどと感じたのなら、このように退化させることをお勧めします。

提案された内容から大部分削ってしまい、`fizz`を返す1行のみにしてしまいます。

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

なんと、補完を受け入れてるうちに大文字小文字の違いを見逃してしまいました。TDDのプラクティスの1つに「**2つのことを同時にするな**」と言うのがあります。TDDだと最初にリストを書き出す時に実現したいことにちゃんと集中しているため、リストを元に書いたテストコード側でこの手のミスが起こりづらいです。TDDではなく実装からはじめてテストを後追いで書いてたら、`fizz`のまま実装もテストも進んでしまったかもしれません。

このようにGitHub Copilotの提案が完全でなくともTDDに則って丁寧に進めてるからこそ救われるケースも多々あるということも、筆者がGitHub CopilotとTDDが相性がいいと考える理由の1つです。

これはプロダクションコードの戻り値を`Fizz`にすればもちろんテストが通ってGREENになります。

```shell-session
3の倍数の場合はFizzを返す ... ok (0ms)

ok | 1 passed | 0 failed (1ms)
```

まだリファクタリングするほどプロダクションコードがないので、TODOリストにチェックを入れて次のテストに進みます。

### 2週目: Buzz

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

### 3週目: FizzBuzz

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

ここで初めてリファクタリングを行います。先頭の分岐は「`15`で割り切れる場合」に変更しても問題ないはずです。ついでに`return`しかないブロックも1行にまとめてしまいます。個人的にはブロックにしておきたい派ですがわかりやすいリファクタだと思うので今回は削ります。このようなリファクタリングも途中でGitHub Copilotが意図を汲み取って補完してくれる様子がみて取れます。

![fizzbuzz refactor 1](/images/tdd-with-copilot/fizz-buzz/refactor-1.png)

リファクタリング後もテストは引き続きGREENです。

### 4週目: それ以外の場合

最後に「それ以外の場合はそのままの数値を返す」のテストを書きます。

![other test](/images/tdd-with-copilot/fizz-buzz/other-test-1.png)

ここに来てついに、**テストケース名まで含め書きたいコードが全て補完**されるようになりました。GitHub Copilotを使ってなくとも、きっと筆者は1文字違わずこのようなテストを書いたことでしょう。**頭の中にあるコードがそのまま出てきた**に等しい体験です。このように学習を重ねることで、GitHub Copilotの提案の質はどんどん向上していきます。

これももちろんテストは失敗するので、プロダクションコードを修正します。

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

### 5週目: テストケースの整理

実際にはFizzBuzzの仕様自体は簡単なのでテストもこれで十分と言えるかもしれませんが、「3の倍数の場合はFizzを返す」は3以外検証してなかったり、少々心許ないのでこれらをグルーピングしながら整理してみます。

![more test](/images/tdd-with-copilot/fizz-buzz/test-refactor-1.png)

これも途中まで書けばGitHub Copilotが意図を汲み取って大部分を補完してくれることでしょう。2つ目のdescribeからは一気に書きたかったテストケースを3つづつ補完してくれたのでさくっとテストケースを追加できました。

```shell-session
3の倍数の場合はFizzを返す ...
  引数: 3 ... ok (1ms)
  引数: 6 ... ok (0ms)
  引数: 9 ... ok (0ms)
3の倍数の場合はFizzを返す ... ok (1ms)
5の倍数の場合はBuzzを返す ...
  引数: 5 ... ok (0ms)
  引数: 10 ... ok (0ms)
  引数: 20 ... ok (0ms)
5の倍数の場合はBuzzを返す ... ok (0ms)
3と5の倍数の場合はFizzBuzzを返す ...
  引数: 15 ... ok (0ms)
  引数: 30 ... ok (0ms)
  引数: 45 ... ok (0ms)
3と5の倍数の場合はFizzBuzzを返す ... ok (0ms)
それ以外の場合はそのままの数値を返す ...
  引数: 1 ... ok (0ms)
  引数: 2 ... ok (0ms)
  引数: 4 ... ok (0ms)
それ以外の場合はそのままの数値を返す ... ok (0ms)

ok | 4 passed (12 steps) | 0 failed (4ms)
```

しかし書き終わってみると少々冗長に感じたので、やはりここは丁寧にパラメータ化テストで書き直してみます。

![more test2](/images/tdd-with-copilot/fizz-buzz/test-refactor-2.png)

既存のテストを書き直してることをしっかり汲み取ってくれています。上記補完後に重複するケースを削除していきます。

他のケースも整理＋少々ケース追加もしてますが、パラメータ化する前と同様のテストを実施できています。

```shell-section
3の倍数の場合はFizzを返す ...
  引数: 3 ... ok (0ms)
  引数: 6 ... ok (0ms)
  引数: 9 ... ok (0ms)
3の倍数の場合はFizzを返す ... ok (0ms)
5の倍数の場合はBuzzを返す ...
  引数: 5 ... ok (0ms)
  引数: 10 ... ok (0ms)
  引数: 20 ... ok (0ms)
5の倍数の場合はBuzzを返す ... ok (0ms)
3と5の倍数の場合はFizzBuzzを返す ...
  引数: 15 ... ok (0ms)
  引数: 30 ... ok (0ms)
  引数: 45 ... ok (0ms)
3と5の倍数の場合はFizzBuzzを返す ... ok (0ms)
それ以外の場合はそのままの数値を返す ...
  引数: 1 ... ok (0ms)
  引数: 2 ... ok (0ms)
  引数: 4 ... ok (0ms)
  引数: 7 ... ok (0ms)
  引数: 8 ... ok (0ms)
それ以外の場合はそのままの数値を返す ... ok (0ms)

ok | 4 passed (14 steps) | 0 failed (2ms)
```

テストもある程度まとまったのでFizzBuzzの実装としては十分でしょう。実践編は以上です。今回のサンプルの完成系は以下のリポジトリにあります。

https://github.com/AkifumiSato/tdd-example-with-copilot

## まとめ

GitHub Copilotが自分の意図をより正確に汲んでくれるようになる、まるで成長するような体験を感じていただけましたでしょうか？もしかしたら「やはり自分でやってみないとわからない」という方もいるかもしれません。まさにその通りだと思います。自分でやってみないと感じられない感動もあります。ぜひ自分の好きなフレームワーク・言語でFizzBuzzなど何か簡単な題材でいいので、GitHub Copilotと一緒にTDDを実践してみてください。

ChatGPTやGitHub Copilot登場以降、AIツールは急速に進化を続けています。そして本稿で紹介したように、TDDなどの従来からあるプラクティスと掛け合わせることでAIツールが真価を発揮するケースもあります。重要なのは過去の知識が不要になったのではなく、**過去の知識と組み合わせることでより高速な開発速度が実現できる**と言う点です。AIツールはエンジニアリングのプラクティスを過去のものにするものではなく、より強力に発展させることができるツールだと筆者は考えています。ようはAIツールも使い方次第ということです。

本稿を通じて、AIツールを使いこなすことが今後大きな武器になるだろうことを感じていただけたなら幸いです。
