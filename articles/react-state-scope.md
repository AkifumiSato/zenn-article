---
title: "ReactのGlobal Stateとライフタイム"
emoji: "⌚️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: false
---

ReactはじめSPAのStateは大きく2種類、Local State・Global Stateの2種類でおおよそのStateの分類が可能であると考えていました。これに対し会社の先輩から意見をもらって、以下2点に気づきました。

- Global Stateには大きく、**UI State**と**Server State**の2つがある
- Stateには**ライフタイム**（生存期間）が存在し、Global Stateにはスコープ的Globalと**時間的Global**の2つが含まれている

これらを意識すると、自分はStateの実装を結構感覚的にやってしまっていたなと気づいたので、Stateの分類について改めてまとめてみようと思います。Reactで何かしらのStateを実装する時に、本稿の分類が実装の参考になれば幸いです。

## スコープによるStateの分類

まずこれまで自分が認識してたスコープにおけるStateの分類について触れておきます。以下の2種類で分類可能と考えていました。

- **Local State**: コンポーネント単位のState。`useState`によって管理し、コンポーネントがアンマウントされるまで生存する。
- **Global State**: 複数のコンポーネントから利用されうる、もしくはページを跨いで利用するState。APIレスポンスやUIの状態などで、ReduxやRecoil、Context APIなどを通じて管理されることが多い。

RecoilやRedux tool kitなどを使ってると、APIレスポンスとUIの状態を同じライブラリで管理できるのでこのように考えていたわけですが、先述の通りGlobal Stateと一纏めにしてしまっているものは**Global State**と**Server State**に分離できます。分離後の定義は以下です。

- **Local State**: コンポーネント単位のState。`useState`によって管理し、コンポーネントがアンマウントされるまで生存する。
- **UI State**: 複数のコンポーネントから利用されうる、もしくはページを跨いで利用するState。~~APIレスポンスや~~UIの状態などで、ReduxやRecoil、Context APIなどを通じて管理されることが多い。
- **Server State**: APIサーバーからのレスポンスやそのキャッシュ。SWRやReact Queryの内部で管理されてるものなどを指す。

UI Stateはクライアントサイドに閉じたスコープですが、Server Stateはクライアントサイドからサーバーサイドまで含む、非常に大きなスコープのStateです。UI Stateは基本内部実装で完結しますが、Server Stateは当然fetchなどを介すため、UI StateとServer Stateは同じGlobal Stateで括られていましたが実装は大きく異なります。

以下参考記事です。

https://zenn.dev/yoshiko/articles/607ec0c9b0408d

細かい用語は違えど、上記記事でもそれぞれ同じよう分類を行っています。

https://react-community-tools-practices-cheatsheet.netlify.app/state-management/overview/#types-of-state

こちらでは本稿同様3種類の分類か、データカテゴリによる分類（Data,Control/UI,Session,Communication,Locationの5つ）が可能とされています。

https://kentcdodds.com/blog/application-state-management-with-react#server-cache-vs-ui-state

Remixのコントリビュータのブログでは、Local/Globalなどの分類ではなく「全てのStateはUI Stateかサーバーキャッシュの2つに分けられる」と主張しています。

これらの参考記事は非常に勉強になるので時間のある方はそれぞれ一読することをお勧めします。

### State分類とライブラリ

前述の通り、UI StateとServer Stateは実装が大きく異なるのでそれぞれ個別に実装戦略を練ることができます。個人的には以下のような実装（ライブラリ選定）がお勧めです。

- Local State: React.useState
- UI State: Recoil
- Server State: SWR

Recoilでもasync selectorを使えばSWRのようにAPIレスポンスのキャッシュなどを扱うこともできますが、実装がSWRより少々冗長な気がするのでServer Stateに対してそれに特化したライブラリを選定するのがお勧めです。以下は実際に利用する時のコード例です。

```ts
// Recoil
const currentUserID = useRecoilValue(currentUserIDState);
const currentUserInfo = useRecoilValue(userInfoQuery(currentUserID));
const refreshUserInfo = useRefreshUserInfo(currentUserID);

// SWR
const { data: user } = useSWR(['/api/user', userId], fetchWithUserId)
```

利用するコンポーネント内でも上記のように3つのhooksが必要になってきますし、`currentUserIDState`/`userInfoQuery`/`useRefreshUserInfo`の定義も必要になってきます。一方SWRでは1行で利用でき、必要な定義も`userId`と`fetchWithUserId`のみです。

### スコープによるStateの分類問題点

スコープによる分類によって実装分担が明確になりましたが、ここでStateの**ライフタイム**の観点を持って考えてみるとまだ問題が混在してる箇所があります。UI Stateの一部をStorageと同期させる場合や、Historyに関連づける場合です。同じUI Stateを名乗っていてもこれらは生存期間が異なり、個別の実装や関連ライブラリが必要になります。

## ライフタイムによるStateの分類

スコープによって大きく3つに分けられたStateの分類を、ライフタイムでより細分化して実装戦略を考察したいと思います。

先に分類したStateをライフタイムで細分化すると以下のようになります。

- Local State
  - Component unmount
- UI State
  - Javascript memory
  - Browser history
  - Browser storage
- Server State
  - Server

Local StateとServer Stateは対応するものが1つづつですが、UI Stateは3つのライフタイムを含んでいます。

### 1. Component unmount

**Component mount**は文字通りコンポーネントがアンマウントされるまでになります。これは`useState`のみで利用することとほぼ同義なので、先のスコープによる分類で言うところの**Local State**と同義になります。

### 2. Javascript memory

**Javascript memory**はJavascriptのメモリが解放されるまで、つまりSPAにおいては「リロードや離脱が発生するまで」になります。スコープにおける分類の**Global State**の最もベーシックな使い方（単に値をGlobalに持つだけ）がこの分類にあたります。

### 3. Browser history

**Browser history**はブラウザの履歴が破棄されるまで、実装的には[history.push](https://developer.mozilla.org/ja/docs/Web/API/History/pushState) や[replaceState](https://developer.mozilla.org/ja/docs/Web/API/History/replaceState) によって履歴に対してObjectが関連付けられるので、このObjectが破棄されるまでになります。実際にObjectが破棄されるタイミングは以下仕様を確認した限りブラウザの実装によりそうですが、documentが非アクティブなタイミングで破棄されうるようです。

https://triple-underscore.github.io/HTML-history-ja.html#session-history

ちなみにnext.jsだと**内部的に`replaceState`で全て独自のObjectで置き換えられてしまう**ため、実質利用できないようです。

https://github.com/vercel/next.js/blob/fe3d6b7aed5e39c19bd4a5fbbf1c9c890e239ea4/packages/next/shared/lib/router/router.ts#L1432


### 4. Browser storage

**Browser storage**はLocal StorageやSession StorageなどのWeb Storageに保存した場合、つまりこれらがブラウザによって破棄されるまでになります。これも要件としては多いようで、UI Stateの関連ライブラリでStorageと一部同期するようなライブラリは多数存在します。

### 5. Server

最後は**Server**、サーバー側でStateが破棄される（=データベースから削除される）までです。これはもちろんServer Stateと同義です。

### State分類とライブラリ

一部のライフタイムはスコープによる分類と同義なので、対応する実装方針を組むことができます。

- Component unmount: React.useState
- Javascript memory: Recoil
- Server: SWR

一方、UI Stateに含まれているライフタイムのBrowser historyとBrowser storageはRecoilの関連ライブラリなどを必要とする場合があります。

Browser storageなStateの実装は[recoil-persist](https://github.com/polemius/recoil-persist) や[redux-persist](https://github.com/rt2zz/redux-persist) など、要望が多いのか関連ライブラリがよく提供されるのでこれらを利用することで簡単に実装できます。

一方でBrowser historyは僕が知らないだけかもしれませんが、History APIによって履歴のエントリーにStateを同期するようなライブラリはあまり見つかりませんでした。これらは[react-router](https://v5.reactrouter.com/web/api/history) などのState Managementライブラリ以外でサポートされていることの方が多いようです。一方、Next.jsは先述の通り内部で独自のObjectでreplaceしてしまうので、実装自体不可能だったりします。

以下まとめです。

- Component unmount: React.useState
- Javascript memory: Recoil
- Browser history: **自前で実装or不可能**
- Browser storage: recoil-persistなど
- Server: SWR

## ライフタイムの分類から見えてくるSPAの問題点

ここまででBrowser historyなライフタイムStateの実装だけ難易度が高いことはおわかりいただけたかと思います。しかしUX面で言うと、**本来多くのStateはBrowser historyなライフタイムであって欲しい**ものです。例えば「アコーディオンを開いた」というStateはSPAでなかった場合には履歴に紐づくものです。SPAだとブラウザバックしたら全てのアコーディオンが閉じてしまってるような経験がある方もいるでしょうが、これらの望ましい体験としてはやはり「履歴に紐づいて復元される」ことだと考えられます。

しかし現実には復元されないSPAが多いように感じますし、関連ライブラリの少なさなどからも実装難易度が高いのが現状です。Recoilの場合、[recoil-sync](https://recoiljs.org/docs/recoil-sync/introduction/) というライブラリを公式が開発中で、これによりURLに対して一意に状態を保持することが容易になりそうなので、こういったライブラリが増えることを個人的には期待しています。

### 余談: Next.jsのscroll amnesiaのFix PR

Browser historyなライフタイムStateの代表格の1つに、スクロール位置があります。ブラウザバック時などにスクロール位置を復元することを`scroll restoration`と呼び、Next.jsではconfigで`experimental.scrollRestoration = true`にすると、ブラウザバック時にスクロール位置が復元されます。

これがリロード時した後に復元に失敗してしまうのを修正したPRを出しているんですが、なかなかマージされません。

https://github.com/vercel/next.js/pull/36861

おそらくレビューアーの中で優先度が低いのでしょう。。。このPRで議論してることとして、「`history`の`key`を公開したい」という話があります。`key`を公開するとhistoryのエントリを一意に識別することができるので、**Browser historyなStateをNext.jsでも実装できるようになります**。個人的には是非とも欲しい機能なので、賛同いただける方はレビューが早く進むかもしれないので、上記PRにリアクションやコメント下さるとありがたいです🙇‍。

## まとめ

少し長くなってしまいましたが、大きく主張したいこととしては大きく以下のみです。

- Stateはスコープやライフタイムによって分類され、分類ごとに実装戦略を検討するのが良さそう
- Stateごとにあるべきライフタイムを考えて実装しよう

ご覧いただき、ありがとうございました。

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
