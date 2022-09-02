---
title: "fetchをSWCプラグインで置き換える"
emoji: "🤪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["swc", "typescript", "rust"]
published: true
---

SWCのプラグインを触ってみたくて、`fetch`を強制置換するプラグインを作成しました。題材として`fetch`のmockを強制するとかやれるといいかなとか思ったのですが、実用的かと言われると素直に[msw](https://mswjs.io/)使った方がいいのでは？って気持ちの方が強いです。なので本当に単に「作ってみたかっただけ」です。備忘録＋SWCのプラグインに興味ある方に参考になれば幸いです。

今回作ったコードは以下にあるので、より詳細にみたい方は以下を参考にして下さい。

https://github.com/AkifumiSato/swc-plugin-fetch-replace/tree/v0.1.1

## SWC plugins docs

SWCのプラグインのドキュメントは以下です。

https://swc.rs/docs/plugin/ecmascript/getting-started

結構丁寧に書いてるしわかりやすかった印象ですが、最初見たときは若干サンプルが古くて動きませんでした。プルリクして修正したので同じ問題は発生しないはずですが、SWCはパッケージの更新早いですし、情報が古い可能性もあるので注意しましょう。

あとはユニットテストの書き方が若干わかりにくかったので、そこらへんも注意です。`test!`というマクロを利用するのですが、ざっくり書くと以下のように使います。

```rust
test!(
    /* TS|ES指定、ESはDefaultでOK */,
    /* テストの実行コード、サンプル参照 */,
    /* テスト名称 */,
    /* input */,
    /* expect */,
)
```

## プラグインの実装

### 実装準備

:::message
SWCはRustなので、Rust開発環境は構築済みの前提です。
:::

ドキュメントの通り、`swc_cli`をインストールします。

```sh
cargo install swc_cli
```

SWCのプラグインのプロジェクトを作成し、`wasm32-wasi`を`target`に追加しておきます。

```sh
swc plugin new --target-type wasm32-wasi my-first-plugin
# You should to run this
rustup target add wasm32-wasi
```

作成されるプロジェクトにはサンプルコードが既にあるので、それをベースに修正していきましょう。

### 変換するASTを探す

次はASTのどの辺を置き換えたいかサンプルコードから推察します。

https://swc.rs/playground

OutputのViewをASTに変更すると、JSON形式でASTが出力されます。試しに以下のコードを入力するとこんな感じのASTが出力されます。

```ts
const res = await fetch('http://localhost:9999');
```

```json
{
  "type": "Module",
  "span": {
    "start": 0,
    "end": 49,
    "ctxt": 0
  },
  "body": [
    {
      "type": "VariableDeclaration",
      "span": {
        "start": 0,
        "end": 49,
        "ctxt": 0
      },
      "kind": "const",
      "declare": false,
      "declarations": [
        {
          "type": "VariableDeclarator",
          "span": {
            "start": 6,
            "end": 48,
            "ctxt": 0
          },
          "id": {
            "type": "Identifier",
            "span": {
              "start": 6,
              "end": 9,
              "ctxt": 0
            },
            "value": "res",
            "optional": false,
            "typeAnnotation": null
          },
          "init": {
            "type": "AwaitExpression",
            "span": {
              "start": 12,
              "end": 48,
              "ctxt": 0
            },
            "argument": {
              "type": "CallExpression",
              "span": {
                "start": 18,
                "end": 48,
                "ctxt": 0
              },
              "callee": {
                "type": "Identifier",
                "span": {
                  "start": 18,
                  "end": 23,
                  "ctxt": 0
                },
                "value": "fetch",
                "optional": false
              },
              "arguments": [
                {
                  "spread": null,
                  "expression": {
                    "type": "StringLiteral",
                    "span": {
                      "start": 24,
                      "end": 47,
                      "ctxt": 0
                    },
                    "value": "http://localhost:9999",
                    "raw": "'http://localhost:9999'"
                  }
                }
              ],
              "typeArguments": null
            }
          },
          "definite": false
        }
      ]
    }
  ],
  "interpreter": null
}
```

気軽に試せるしわかりやすいしASTやトランスパイル後のコードも見れるし、すごいです。

ここで目的の`fetch`についてASTのJSONを見てみると`fetch`の呼び出しは`CallExpression`の`Identifier`でマッチングすることができそうです。

### `fetch`の置き換え

`fetch`のASTを実際に置き換える前に、置き換え後の関数名を`.swcrc`から`replaceName`として受け取って保持するようにしましょう。利用イメージは以下です。

```json
// .swcrc
{
  "jsc": {
    "experimental": {
      "plugins": [
        [
          "swc-plugin-fetch-replace",
          {
            "replaceName": "replace_fetch"
          }
        ]
      ]
    }
  }
}
```

`replaceName`をkeyに持つオブジェクトの構造体を定義します。

```rust
#[derive(Default, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
#[serde(deny_unknown_fields)]
pub struct Config {
    /// Function name to replace
    #[serde()]
    pub replace_name: String,
}

pub struct TransformVisitor {
    config: Config,
}

impl TransformVisitor {
    pub fn new(config: Config) -> Self {
        Self { config }
    }
}
```

これをプラグインのエンドポイントで受け取って保持します。

```rust
#[plugin_transform]
pub fn process_transform(program: Program, metadata: TransformPluginProgramMetadata) -> Program {
    let config = serde_json::from_str::<Config>(
        &metadata
            .get_transform_plugin_config()
            .expect("failed to parse plugin config"),
    )
    .expect("invalid plugin config");

    program.fold_with(&mut as_folder(TransformVisitor::new(config)))
}
```

`self.config.replace_name`で置き換える関数名を参照できるようになりました。あとは`CallExpression`の`Identifier`にマッチして値が`fetch`の時に、上記関数名に置き換えします。

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use swc_core::testing_transform::test;

    fn test_visiter() -> TransformVisitor {
        let config = Config {
            replace_name: String::from("my_test_fetch"),
        };
        TransformVisitor { config }
    }

    test!(
        Default::default(),
        |_| as_folder(test_visiter()),
        replace_fetch,
        // Input codes
        r#"
        const res = await fetch('http://localhost:9999');
        "#,
        // Output codes after transformed with plugin
        r#"
        const res = await my_test_fetch('http://localhost:9999');
        "#
    );
}
```

上記テストが通るように、`calee`のvisitorである`visit_mut_callee`を修正します。

```rust
impl VisitMut for TransformVisitor {
    fn visit_mut_callee(&mut self, callee: &mut Callee) {
        callee.visit_mut_children_with(self);

        if let Callee::Expr(expr) = callee {
            if let Expr::Ident(i) = &mut **expr {
                if &*i.sym == "fetch" {
                    let replace_name: &str = &self.config.replace_name;
                    i.sym = replace_name.into();
                }
            }
        }
    }
}
```

関数名の名前一致なのでだいぶ乱暴ですが、とりあえずプラグインを作ってみたかったベースなのでOKということにします。

これでテストが通り、置き換えられることが確認できました。

### `window.fetch/globalThis.fetch`の置き換え

`fetch`はグローバルスコープなので、`window.fetch`や`globalThis.fetch`でも呼び出せます。これらも置き換えをやってみます。

```rust
#[cfg(test)]
mod tests {
    // ...省略

    test!(
        Default::default(),
        |_| as_folder(test_visiter()),
        global_this_fetch,
        // Input codes
        r#"
        const res = await globalThis.fetch('http://localhost:9999');
        "#,
        // Output codes after transformed with plugin
        r#"
        const res = await globalThis.my_test_fetch('http://localhost:9999');
        "#
    );

    test!(
        Default::default(),
        |_| as_folder(test_visiter()),
        widow_fetch,
        // Input codes
        r#"
        const res = await window.fetch('http://localhost:9999');
        "#,
        // Output codes after transformed with plugin
        r#"
        const res = await window.my_test_fetch('http://localhost:9999');
        "#
    );
}
```

上記テストが通ればOKです。例によって[playground](https://swc.rs/playground)で以下の実装を入力してみましょう。

```ts
const res = await window.fetch('http://localhost:9999');
```

ASTのJSONは省略しますが、今度は`CallExpression > MemberExpression > Identifier(window | globalThis) > Identifier(fetch)`と深いマッチングが必要になります。以下実装です。

```rust
impl VisitMut for TransformVisitor {
    fn visit_mut_callee(&mut self, callee: &mut Callee) {
        callee.visit_mut_children_with(self);

        if let Callee::Expr(expr) = callee {
            if let Expr::Member(parent) = &mut **expr {
                if let Expr::Ident(i) = &mut *parent.obj {
                    if &*i.sym == "window" || &*i.sym == "globalThis" {
                        if let MemberProp::Ident(i) = &mut parent.prop {
                            if &*i.sym == "fetch" {
                                let replace_name: &str = &self.config.replace_name;
                                i.sym = replace_name.into();
                            }
                        }
                    }
                }
            }

            if let Expr::Ident(i) = &mut **expr {
                if &*i.sym == "fetch" {
                    let replace_name: &str = &self.config.replace_name;
                    i.sym = replace_name.into();
                }
            }
        }
    }
}
```

綺麗に波動拳です。（ASTいじる時ってどうしてもこうなりますよね...？）

これでまたテストが通り、無事`window.fetch`や`globalThis.fetch`の置き換えもできるようになりました。

## publish

ついでなんで`npm publish`してみます。以下コマンドでbuildしてリリース物を用意します。

```sh
cargo build-wasi --release && cp target/wasm32-wasi/release/swc_plugin_fetch_replace.wasm .
```

ファイルが用意できたら適宜`package.json`のバージョンやmainファイルなどを修正します。

```json
{
    "name": "swc-plugin-fetch-replace",
    "version": "0.1.1",
    "description": "SWC plugin for replaces global `fetch` with an arbitrary function.",
    "author": "akfm.sato@gmail.com",
    "license": "MIT",
    "keywords": ["swc-plugin"],
    "main": "swc_plugin_fetch_replace.wasm",
    "scripts": {
        "prepack": "cargo build-wasi --release && cp target/wasm32-wasi/release/swc_plugin_fetch_replace.wasm ."
    },
    "files": ["swc_plugin_fetch_replace.wasm"]
}
```

あとは`npm publish`するだけです。

```sh
npm publish
```

無事publishできました。

https://badge.fury.io/js/swc-plugin-fetch-replace

## 感想

せっかくpublishしたしjestで使ってみたんですが、やはり普通にmsw使った方が便利です...w

でもまぁ、とりあえず意図した通りに動いてるし一通りやってみると理解が進んで勉強になりました。
