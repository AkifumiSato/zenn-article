---
title: "react-hook-formとformの設計原則"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "reacthookform"]
published: false
---

reactでformを作る時、[react-hook-form](https://react-hook-form.com/)を使う方も多いと思います。react-hook-fomで実装すると、非制御コンポーネントベースなためレンダリングコストを減らすことが期待でき（実装にもよります）、また、[zod](https://zod.dev/)との連携によるバリデーション実装の容易さなど、実装や設計面においても多くのメリットが得られます。

一方で、チームでreact-hook-fomを使って実装を進めていくには設計的難しさを伴うことがあります。筆者は実際に以下2つのケースで設計の見直しを迫られました。

- form内におけるモーダルの実装方法
- 保存ボタンの移動

本稿はこれらの課題に直面し、設計を見直すことで筆者が感じたreact-hook-fomを使った実装における設計原則のまとめになります。

:::message
本稿ではreact-hook-fomの初歩的な実装や基本的なAPIについては解説しないので、まだ触ったことがない・自信がないという方は[公式チュートリアル](https://react-hook-form.com/get-started)など参照することをお勧めします。
:::

## form実装の3原則

先に結論です。react-hook-fomを使ったform実装においては以下の3つの原則を守ることが設計上重要になってきます。

1. form実装はreact-hook-fomで統一する
2. 選択モーダルは1つのformとして扱う
3. 保存処理は常に分離を意識する

これらの原則について、1つづつ詳解していきます。

## 1. form実装はreact-hook-fomで統一する

筆者が最も重要だと考えているのが、**form実装はreact-hook-fomで統一する**ということです。チームで実装を進めるにあたり、設計のシンプルさは非常に重要です。form周りの実装でreact-hook-fomだったりRecoilだったりuseStateだったり、コンポーネントによって実装がさまざまだと、チームで統一された設計思想を持つことが難しくなります。そして、統一した設計思想をチームで維持できない場合、大幅な手戻りにつながる可能性があります。

しかしながら言うは易く行うは難し、現実にはreact-hook-fomでは実装しづらい要件に直面することもあります。筆者が経験した中で最も顕著だったのが、前述の通りformにおけるモーダルの実装です。

## 2. 選択モーダルは1つのformとして扱う

formにおける選択モーダルにはいくつかの仕様パターンが想定されます。

1. **保存ボタンなどを押下しないとformに反映されない**
1. 保存ボタンなどを押下しないとformに反映されないが、保存ボタンでしかモーダルを閉じれない
1. 入力が即時にformに反映される

もちろんUIにもよるのですが、筆者が最も基本的で好ましいと感じるのは1番目の仕様です。

:::details demo
![](/images/principles-of-form/form-modal-demo.gif)
:::

2,3番目の仕様ならreact-hook-formで実装するのにそこまで困らないことでしょう。ただし、1番目の仕様を実装しようとすると一筋縄ではいきません。保存ボタンを押下するまで反映しないようにする以上、ページComponentで宣言・取得した`register`は選択モーダルの入力要素には利用できません。保存を押す前から値がformに反映されてしまうからです。

```tsx
<form
  onSubmit={handleSubmit((data) => {
    console.log("submit: ", data);
  })}
>
  {/* ...ページのform、大部分は省略... */}
  <Modal
    closeButton
    aria-labelledby="modal-title"
    open={visible}
    onClose={closeHandler}
  >
    <Modal.Header>
      <Text id="modal-title" size={15}>
        クレジットカード番号を入力してください
      </Text>
    </Modal.Header>
    <Modal.Body>
      <Input
        clearable
        bordered
        fullWidth
        color="primary"
        size="lg"
        placeholder="xxxx-yyyy-zzzz"
        {...register("credit")}
      />
    </Modal.Body>
    <Modal.Footer>
      <Button auto onPress={closeHandler} type="submit">
        保存
      </Button>
    </Modal.Footer>
  </Modal>
  <Button type="submit">submit</Button>
</form>
```

このモーダル内の値の保持だけ`useState`や`Recoil`などで実装すれば良いのでは、と思った方もいらっしゃるかと思います。しかし、これは1つ目の原則に違反しており、react-hook-fomのバリデーションを利用できず、react-hook-formが提供してる機能を一部自前で実装する必要が出てきてしまうなどのデメリットが伴います。

このような場合、筆者は**選択モーダルは1つのformとして扱う**ことで、react-hook-fomに実装を集約します。以下は簡略化してますが、実装例です。

```tsx
// Home.tsx
export default function Home() {
  const { register, handleSubmit, setValue, watch } = useForm<{
    name: string;
    credit: string;
  }>();
  const credit = watch("credit");
  const [visible, setVisible] = useState(false);
  const closeHandler = () => setVisible(false);

  return (
    <>
      <Head>
        <title>Create Next App</title>
        <meta name="viewport" content="width=device-width, initial-scale=1" />
      </Head>
      <main className={styles.main}>
        <div className={styles.wrapper}>
          <Text h1 size={30} weight="bold">
            サンプルForm
          </Text>
          <form
            onSubmit={handleSubmit((data) => {
              console.log("submit: ", data);
            })}
          >
            <div className={styles.formInner}>
              <dl className={styles.formItem}>
                <dt>名前</dt>
                <dd>
                  <Input
                    clearable
                    initialValue="田中太郎"
                    width="200px"
                    {...register("name")}
                  />
                </dd>
              </dl>
              <dl className={styles.formItem}>
                <dt>クレジットカード</dt>
                <dd className={styles.formItemContents}>
                  <Text>{credit ?? "none"}</Text>
                  <Button onClick={() => setVisible(true)} auto flat>
                    変更
                  </Button>
                </dd>
              </dl>
              <Button type="submit">submit</Button>
            </div>
          </form>
          <Modal
            closeButton
            aria-labelledby="modal-title"
            open={visible}
            onClose={closeHandler}
          >
            <CreditModal
              closeHandler={closeHandler}
              onSubmit={({ credit }) => {
                console.log("CreditModal submit");
                setValue("credit", credit);
              }}
            />
          </Modal>
        </div>
      </main>
    </>
  );
}

// CreditModal.tsx
import { Input, Text, Button, Modal, Row } from "@nextui-org/react";
import { useForm } from "react-hook-form";

type Props = {
  closeHandler: () => void;
  onSubmit: (data: { credit: string }) => void;
};

export default function CreditModal({ closeHandler, onSubmit }: Props) {
  const { handleSubmit, register } = useForm<{ credit: string }>();
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Modal.Header>
        <Text id="modal-title" size={15}>
          クレジットカード番号を入力してください
        </Text>
      </Modal.Header>
      <Modal.Body>
        <Input
          clearable
          bordered
          fullWidth
          color="primary"
          size="lg"
          placeholder="xxxx-yyyy-zzzz"
          {...register("credit")}
        />
      </Modal.Body>
      <Modal.Footer>
        <Button auto onPress={closeHandler} type="submit">
          保存
        </Button>
      </Modal.Footer>
    </form>
  );
}
```

`Home`と`CreditModal`でそれぞれ`useForm`しており、`CreditModal`のsubmit時に`Home`のformに`setValue`しています。

```tsx
onSubmit={({ credit }) => {
  console.log("CreditModal submit");
  setValue("credit", credit);
}}
```

### formとはなんなのか

ここで画面内に複数のformを持つことの是非について考えてみましょう。これは「」と言う問いについて考えることと同義です。筆者はこれについて、「**formとは1つのトランザクションである**」と考えています。ここで言うトランザクションはSQLのトランザクションと同様に、commitかrollbackされるまで反映されないが入力を保持する一連の処理を指します。

モーダルの仕様は「保存ボタンなどを押下しないとformに反映されない」でした。これは1つのトランザクションであり、1つのformと見なすことができます。画面埋め込みの場合も、一連の処理がcommitされたらサーバーへデータを送信するトランザクションと見なすことができます。

また、モーダル内で1つのformを持つことはreactなどを利用していない場合も、formタグの[`method="dialog"`](https://developer.mozilla.org/ja/docs/Web/HTML/Element/form#attr-method)を使えばかなり小さい実装で実現可能です。実際最近多くのブラウザでサポートされている[`dialog`要素の実装例](https://developer.mozilla.org/ja/docs/Web/HTML/Element/dialog#%E5%BF%9C%E7%94%A8%E7%9A%84%E3%81%AA%E4%BE%8B)では`dialog`内に1つのformタグを利用し、`method="dialog"`で`form`の外側へデータを受け渡すようになっています。

:::details MDNの実装例の引用
```html: html
<dialog id="favDialog">
  <form method="dialog">
    <p><label>Favorite animal:
      <select>
        <option></option>
        <option>Brine shrimp</option>
        <option>Red panda</option>
        <option>Spider monkey</option>
      </select>
    </label></p>
    <menu>
      <button value="cancel">Cancel</button>
      <button id="confirmBtn" value="default">Confirm</button>
    </menu>
  </form>
</dialog>

<menu>
  <button id="updateDetails">Update details</button>
</menu>

<output aria-live="polite"></output>
```

```js: js
var updateButton = document.getElementById('updateDetails');
var favDialog = document.getElementById('favDialog');
var outputBox = document.querySelector('output');
var selectEl = document.querySelector('select');
var confirmBtn = document.getElementById('confirmBtn');

// "Update details" button opens the <dialog> modally
updateButton.addEventListener('click', function onOpen() {
  if (typeof favDialog.showModal === "function") {
    favDialog.showModal();
  } else {
    alert("The <dialog> API is not supported by this browser");
  }
});
// "Favorite animal" input sets the value of the submit button
selectEl.addEventListener('change', function onSelect(e) {
  confirmBtn.value = selectEl.value;
});
// "Confirm" button of form triggers "close" on dialog because of [method="dialog"]
favDialog.addEventListener('close', function onClose() {
  outputBox.value = favDialog.returnValue + " button clicked - " + (new Date()).toString();
});
```
:::

実際、モーダルを1つのformとして扱うことであとでモーダルをページにしたり、逆にページをモーダルにするのに苦労せずに済みます。formとformの繋ぎ込みはreact-hook-formnの場合、すでに挙げたようにsubmit時に`setValue`するだけです。

## 3. 保存処理は常に分離を意識する

3番目の原則は**保存処理は常に分離を意識する**ことです。冒頭で実際に設計の見直しを迫られた事情として、「保存ボタンの移動」を挙げました。保存ボタンの要素はform内に配置することになるはずですが、formのページで保存ボタンの位置を変えることは往々にして考えられます。元はなかったが画面固定にしたい、画面上部にも置きたい、ページによってはモーダル内・画面内などいろいろなケースが考えられます。

こういった変更を行うときに保存ボタンがformに密結合な場合、移動したいだけなのに修正量が多くなったり、筆者のように設計の見直しを迫られることになります。これはSOLID原則の単一責務の原則（変更する理由は1つであるべき）に近いかもしれません。保存ボタンの位置を変更したいだけなのに、formタグの位置からして見直さないといけなくなったりする状態はわかりやすく修正難易度を高めてしまいます。

## まとめ

EFO、エントリーフォーム最適化（Entry Form Optimization）という言葉があるように、formの実装は複雑かつビジネス価値の高い部分であることが多いUIです。そして複雑かつビジネス価値が高いからこそ、高い変更頻度を伴うことがあったり、堅牢な修正が求められます。

react-hook-formは素晴らしいライブラリです。自由度が低く、仕様によっては実装しづらかったりもしますが、だからこそシンプルに保とうという意識が実装者側に芽生えやすいのも、個人的には気に入ってる理由の1つです。複雑なものをいかに実装させないかというのは大事な観点です。

しかしreact-hook-formを持ってしても、やはり設計が難しいのがformです。この記事がformの実装設計で悩む方の参考になれば幸いです。

## todo

- 全体見直し
  - 原則ごとの分量が異なるのを調整したい
- 単語の整理と定義
