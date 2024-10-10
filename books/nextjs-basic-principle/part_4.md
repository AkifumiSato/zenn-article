---
title: "第4部 レンダリング"
---

従来Pages Routerは[SSR](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering)・[SSG](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation)・[ISR](https://nextjs.org/docs/pages/building-your-application/data-fetching/incremental-static-regeneration)という3つのレンダリングモデルをサポートしてきました。App Routerは引き続きこれらをサポートしていますが、これらに加え**Streaming**に対応している点が大きく異なります。

具体的には[Streaming SSR](https://nextjs.org/docs/app/building-your-application/rendering/server-components#streaming)や[Partial Pre-Rendering(PPR)](https://nextjs.org/docs/app/api-reference/next-config-js/partial-prerendering)など、Pages Routerにはない新たなレンダリングモデルがサポートされています。特にPPRは、従来より高いパフォーマンスを実現しつつもシンプルなメンタルモデルでNext.jsを扱うことができるようになります。

第4部ではReactやApp Routerにおけるレンダリングの考え方について解説します。
