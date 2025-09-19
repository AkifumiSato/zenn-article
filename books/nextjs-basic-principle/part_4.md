---
title: "第4部 レンダリング"
---

従来Pages Routerは[SSR↗︎](https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering)・[SSG↗︎](https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation)・[ISR↗︎](https://nextjs.org/docs/pages/building-your-application/data-fetching/incremental-static-regeneration)という3つのレンダリングモデルをサポートしてきました。App Routerは引き続きこれらをサポートしていますが、これらに加え[Streaming SSR↗︎](https://nextjs.org/docs/app/getting-started/linking-and-navigating#streaming)に対応している点が大きく異なります。

第4部ではReactやNext.jsにおけるレンダリングの考え方について解説します。
