---
title: "Custom Next.js Cache Handler
 - Vercel以外でのNext.jsキャッシュ活用"
emoji: "💸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

## 構成

- 序文
  - Next.jsで実装したアプリケーションのデプロイ先としてVercelはとても利便性が高く、優れたプラットフォーム
  - 一方インフラ的な都合やコスト的な都合でSelf hosting、つまりVercel以外を利用したいケースはそれなりに多いのではないでしょうか
  - 筆者や筆者の周りではSelf hostingでNext.jsを利用している方は多く見受けられます
  - Next.jsはVercel非依存なOSSと銘打ってるが、実際にはVercelが隠蔽してるインフラ的仕様を実現するのは、高いハードルです
  - 昨今、App Routerの登場とその強力なキャッシュ戦略により、よりVercel以外でNext.jsを扱うことは難しくなってきました
  - 一方で、Self hosting向けのドキュメントや対応は少しづつであるが取り組みがなされています
  - 今回はその一貫で出てきた、`cache-handler`を利用してNext.jsのキャッシュをRedisに保存する方法について紹介します
- Next.jsのキャッシュ
  - Next.jsには複数のキャッシュが存在します
  - Pages RouterではISRを利用しなければあまり気にしなくてもよかったのですが、App Routerにおいては、4種類（一時期は5種類目の検討も示唆されていた）存在します
  - これらNext.jsのキャッシュは、デフォルトではファイルキャッシュとなっています
  - 昨今のインフラ構成においてファイルキャッシュは、少々勝手が悪いものといえます
  - AWSではEFSなどを用いてファイルを共有するなどが考えられますが、ファイルI/Oの競合なども考えられるので容易ではありません
  - 代替策として、キャッシュ永続用のRedisなどを用意することが考えられます
  - しかしこれまでは、Next.jsのキャッシュ永続化を設定するオプションが存在しませんでした
- Custom Next.js Cache Handlerとは
  - `Custom Next.js Cache Handler`は、Next.jsのキャッシュ永続化先をカスタマイズできるオプションで、最近のSelf hostingサポートのDiscussionなどを経て出てきた機能です
  - この機能を利用するには、Next.jsのin memoryキャッシュを無効化することが必須です
    - 4種類のキャッシュには書かれてないが、実はFull Route CacheやData Cacheにはファイルキャッシュを読み込んだ際にin memoryに保存する内部キャッシュまで存在します
  - cacheのハンドリング処理自体はAPI仕様に基づいて実装することでカスタマイズできます
  - 現在は3つのメソッド（`get`/`set`/`revalidateTag`）を実装することで、キャッシュの読み書きなどの実装が可能です
    - `revalidateTag`のみで`revalidatePath`がないのは、`revalidatePath`は内部的に`revalidateTag`を呼び出しており実質的にwrapperであるためです
  - しかしこれを自前で実装するのはRedisや`ioredis`などの知識も必要となり、少々骨が折れる作業です
- `neshca`とは
  - そこで公式のexamplesでは、`neshca`というRedisを利用したCustom Next.js Cache Handlerの実装が提供されています
  - `neshca`はNext.js Shared Cacheの略です
  - Vercelとの直接的な関係は確認できてないので、おそらく有志による開発だと思われます
  - これを使うことで、Redisに対するキャッシュの読み書きがかなり簡単に実装できます
  - （サンプル）
- まとめ
  - `neshca`や`Custom Next.js Cache Handler`はまだ登場したばかりのため、プロダクションでの実運用においてどういう問題が起きるかなどについては未知数です
  - 一方、これまでSelf hosting向けのサポートは手薄く感じていたので、こういう機能が出てきたことは大きな進展とも考えています
  - 実際、これはVercelにとっては不利な機能開発なはずです
  - それでもこういう機能が出てきたのは、App Routerの普及にはSelf hostingのサポート需要が無視できないものだったということなのかもしれません
  - 今後Self hosting向けのNext.jsの機能開発がより進展することを期待したいところです

## todo

- 簡単なサンプルを実装する
  - Redis(not stack)で構築する
  - git hashをprefixにする
    - https://caching-tools.github.io/next-shared-cache/configuration/build-id-as-prefix-key
  - TTL設定
