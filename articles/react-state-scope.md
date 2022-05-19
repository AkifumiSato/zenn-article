---
title: "ReactのGlobal Stateとライフタイム"
emoji: "⌚️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: false
---

ReactはじめSPAのStateは大きく2種類、Local State・Global Stateの2種類でおおよそのStateの分類が可能であると考えていました。これに対し会社の先輩から意見をもらって、以下2点に気づきました。

- Global Stateには大きく、**UI State**と**Server State**の2つがある
- Global Stateにはスコープ的Globalな意味合いと時間的Globalな意味合いが含まれている

なんとなくで実装してしまってたのでこの3種類のStateの分類を整理し、さらにこの値の生存期間をRustから言葉を借りてStateの**ライフタイム**と呼称し、ライフタイム別にStateを再分類しState戦略について自分なりに考察したいと思います。

Reactで何かしらのStateを実装する時に、本稿の分類が実装の参考になれば幸いです。

:::message
Rustのライフタイムはコンパイラによってチェックされる機能名を兼ねていますが、ここではスコープとは別の分類軸のための概念として表記しています。
:::

## スコープによるStateの分類

まずこれまで自分が認識してた参照スコープにおけるStateの分類について触れておきます。以下の2種類で分類可能と考えていました。

- **Local State**: コンポーネント単位のStateで、`useState`で管理します。コンポーネントがアンマウントされるまで生存します。
- **Global State**: 複数のコンポーネントから利用されうる、もしくはページを跨いで利用するState。APIレスポンスやUIの状態などで、ReduxやRecoil、Context APIなどを通じて管理されることが多いかと思います。

RecoilやRedux tool kitなどを使ってると、APIレスポンスとUIの状態を同じライブラリで管理できるのでこのように考えていたわけですが、本稿の1つ目の主題は先述の通り、Global Stateと一纏めにしてしまっているものが**Global State**と**Server State**に分離できるということです。分離後の定義は以下です。

- **Local State**: コンポーネント単位のState。`useState`で管理します。コンポーネントがアンマウントされるまで生存します。
- **UI State**: 複数のコンポーネントから利用されうる、もしくはページを跨いで利用するState。~~APIレスポンスや~~UIの状態などで、ReduxやRecoil、Context APIなどを通じて管理されることが多いかと思います。
- **Server State**: クライアントサイドでキャッシュしてるAPIサーバーからのレスポンス値などです。SWRやReact Queryの内部で管理されてるものなどを指します。捉え方次第では、APIに保存されている状態も同様にServer Stateと見なすこともできます。

以下参考記事です。

https://zenn.dev/yoshiko/articles/607ec0c9b0408d

細かい用語は違えど、上記記事でもそれぞれ同じよう分類を行っています。

https://react-community-tools-practices-cheatsheet.netlify.app/state-management/overview/#types-of-state

こちらでは本稿同様3種類の分類か、データカテゴリによる分類（Data,Control/UI,Session,Communication,Locationの5つ）が可能とされています。

https://kentcdodds.com/blog/application-state-management-with-react#server-cache-vs-ui-state

Remixのコントリビュータのブログでは、Local/Globalなどの分類ではなく「全てのStateはUI Stateかサーバーキャッシュの2つに分けられる」と主張しています。

これらの参考記事は非常に勉強になるので時間のある方はそれぞれ一読することをお勧めします。

### State分類とライブラリ

ここで主張したいこととしては、UI StateとServer Stateは同じライブラリで扱うこともできますが、**それぞれ用途に応じて別々に選択するという手段もある**ということです。具体的には

- Local State: React.useState
- UI State: Recoil
- Server State: SWR

というような選択などです。Recoilでもasync selectorを使えばSWRのようにAPIレスポンスのキャッシュなどを扱うこともできますが、技術選定としてはそれぞれ用途ごとに分けるという選択肢もあるということです。個人的には上記例にあげたように、RecoilとSWRを併用して実装するのがおすすめです。Recoilのasync selectorは実装がSWRより少々冗長な気がするのと、キャッシュをpurgeしようと思った時に依存するatomsを更新するなどするのがまた冗長な気がするからです。以下は実際に利用する時のコード例です。

```ts
// Recoil
const currentUserID = useRecoilValue(currentUserIDState);
const currentUserInfo = useRecoilValue(userInfoQuery(currentUserID));
const refreshUserInfo = useRefreshUserInfo(currentUserID);

// SWR
const { data: user } = useSWR(['/api/user', userId], fetchWithUserId)
```

利用するコンポーネント内でも上記のように3つのhooksが必要になってきますし、`currentUserIDState`/`userInfoQuery`/`useRefreshUserInfo`の定義も必要になってきます。一方SWRでは1行で利用でき、必要な定義も`userId`と`fetchWithUserId`のみです。

### スコープによるStateの分類まとめ

このように、Stateを3つに分けて考えることでそれぞれ役割に適したライブラリ選定を行う余地が生まれます。もちろん、必ずしもUI StateとServer Stateでライブラリ選定を分ける必要はありませんが、それでも「最初からまとめて検討する」より「State分類ごとに検討する」の方が選択の幅が広がります。

また、一言にGlobal Stateと言っても人によってUI StateかServer Stateどちらかに主軸を置いて考えてしまうこともあるので、共通認識が取りやすいという面でもこの3つの分類に分けて会話するメリットがあるんじゃないかと思います。

## ライフタイムによるStateの分類

次にライフタイムによるStateの分類ですが、スコープによるStateの分類は「**どういう状態があってどう実装するのか**」という話でしたが、ライフタイムによるStateの分類は「**どういう生存期間があってどう実装するのか**」という話になります。

UI Stateはページを跨いで保存するようなものも含まれると定義しましたが、ページ遷移を跨いで保存する方法は1つではありません。例えば「アコーディオンを開いた」というStateは**SPAでなかった場合には履歴に紐づくもの**です。SPAだとブラウザバックしたら全てのアコーディオンが閉じてしまってるような経験がある方もいるでしょうが、これらの望ましい体験としてはやはり「履歴に紐づいて復元される」ことだと考えられます。JSで扱ってるStateを履歴に紐づけるには[history.push](https://developer.mozilla.org/ja/docs/Web/API/History/pushState) や[replaceState](https://developer.mozilla.org/ja/docs/Web/API/History/replaceState) を利用してStateを保存する必要があります。

このように、Stateには履歴に紐づく生存期間が適切なものや、コンポーネントが破棄されたらStateも破棄してよいようなものまでいくつかの分類が存在します。

## ライフタイムによるStateの分類

Stateはライフタイムによって大きく以下5つに分類できます。

### 1. Component unmount

1つ目のライフタイム分類は**Component mount**、文字通りコンポーネントがアンマウントされるまでになります。これは`useState`のみで利用することとほぼ同義なので、先のスコープによる分類で言うところの**Local State**になります。

### 2. Javascript memory

**Javascript memory**はJavascriptのメモリが解放されるまで、つまりSPAにおいては「リロードや離脱が発生するまで」になります。スコープにおける分類の**Global State**の最もベーシックな使い方（単に値をGlobalに持つだけ）がこの分類にあたります。

### 3. Browser history

**Browser history**はブラウザの履歴が破棄されるまで、実装的には[history.push](https://developer.mozilla.org/ja/docs/Web/API/History/pushState) や[replaceState](https://developer.mozilla.org/ja/docs/Web/API/History/replaceState) によって履歴に対してObjectが関連付けられるので、このObjectが破棄されるまでになります。実際にObjectが破棄されるタイミングは以下仕様を確認した限りブラウザの実装によりそうですが、documentが非アクティブなタイミングで破棄されうるようです。

https://triple-underscore.github.io/HTML-history-ja.html#session-history

ちなみにnext.jsだと内部的に`replaceState`で全て独自のObjectで置き換えられてしまうため、実質利用できないようです。

https://github.com/vercel/next.js/blob/fe3d6b7aed5e39c19bd4a5fbbf1c9c890e239ea4/packages/next/shared/lib/router/router.ts#L1432

----- ここまでみた -----

### 4. Browser storage

- session
- local

### 5. Server

## State Managementライブラリを分類

## まとめ

## 参考

### Rustにおけるライフタイム

Stateの再分類に入る前に、単語の借用元であるRustのライフタイムについても軽く触れておきます。興味のある方はRustの[The Book](https://doc.rust-jp.rs/book-ja/ch10-03-lifetime-syntax.html) と呼ばれる入門サイトがあるので、こちらを参照ください。

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

`&x`は変数`x`の参照を表しており、このエラーは変数`x`の生存期間が短すぎる旨を表しています。Rustでは変数がスコープを抜ける時にその変数のメモリを解放します。解放したメモリに対する参照が残ってしまうと、簡単にバグの温床となるでしょう。Rustではこれを防ぐために、コンパイラが変数にライフタイムという注釈をつけて生存期間を管理しているわけです。先の例のライフタイムは以下のように、変数`r`には`'a`、変数`x`には`'b`というライフタイム注釈が付与されて、`'b`の枠を超えて`x`は生存できない決まりになっています。

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

関数の戻り値などでライフタイムをコンパイラに明示するために、`<'a>`のようにライフタイムを明示することもできますが、本稿では触れないので気になる方は前述の[The Book](https://doc.rust-jp.rs/book-ja/ch10-03-lifetime-syntax.html) を参照ください。

- Scope of State
  - 2種類だと思ってた＋なんとなくでやってしまってた
  - 3種類目、サーバーState
- Lifetime of State
  - Global Stateには時間的Globalなものが含まれている
    - これは技術選定より、仕様の定義に近い
  - 仕様的にこの値はいつまで生存すべきなのかを意識する必要がある
    - アコーディオン開いて遷移して戻ったら開いてて欲しいよね？
    - でも開いて回遊して戻ってきて開いてるってなんか不思議
  - Stateをライフタイムで再分類してみる
  - 再分類した上で、最適と考える実装例
    - いつRecoilなのか、いつuseStateなのか、いつどうやって実装するのか
  - Historyに紐づくState管理はNextはしづらい
    - scroll 
    - PR出してる
- まとめ
  - Stateのライブラリ選定や実装戦略はStateを3つに分けて考えると良い
  - あるStateの生存期間は5つのlifetimeに分けてどれに当てはめるのが最適か考えると仕様を決めやすい
