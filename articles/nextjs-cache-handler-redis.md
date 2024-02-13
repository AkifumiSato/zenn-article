---
title: "Vercel以外でのNext.jsキャッシュ活用入門"
emoji: "💸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

## 構成

- 序文
  - インフラ的な都合やコスト的な都合などで、Next.jsをVercel以外で使いたいという方は多いのではないだろうか
  - 筆者もそうだが、勉強会などでもよく聞くケースである
  - Next.jsはVercel非依存なOSSと銘打ってるが、実際にはVercelが隠蔽してるインフラ的仕様を実現するのは、高いハードルである
  - 昨今、App Routerの登場とその強力なキャッシュ戦略により、よりVercel以外でNext.jsを扱うことは難しくなってきた
  - 一方で、Self hosting向けのドキュメントや対応は少しづつであるが取り組みがなされてる
  - 今回はその一貫で出てきた、`cache-handler`によるNext.jsのキャッシュをRedisに保存する方法について紹介する
- Next.jsのキャッシュのデフォルト挙動
- cache-handlerとは
- Redis、Redis-Stackとは
- `neshca`とは

## todo

- cache handlerのドキュメント読む
