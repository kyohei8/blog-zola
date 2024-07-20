+++
title="chakra-uiでNext.jsのLinkを利用するとき"
[taxonomies]
tags=["ChakraUI", "Next.js"]
+++

### The TL;DR:

Next.js の Link component と chakraUI の Link component を利用するときは `hrefPass` を利用します。

```ts
import NextLink from 'next/link';
import { Link } from '@chakra-ui/react';

<NextLink href="/home" passHref>
  <Link>Link</Link>
</NextLink>

// chakra.a でも同じ
<NextLink href="/home" passHref>
  <chakra.a>Link</chakra.a>
</NextLink>
```

### なぜなのか

passHref がない場合、html は以下のようになり href が出力されていない状態になります。この場合、クリック時は問題なく動作しますが、Mac でリンク先を別タブで開きたい場合の「command+click」が動作しなくなります。

```ts
<a class="chakra-link css-f4h6uy">Link</a>
```

hrefPass を利用した場合は href が付きます。

```ts
<a class="chakra-link css-f4h6uy" href="/home">
  Link
</a>
```

もちろん、手動で herf を指定してもよい。ただし間違えたりするのでおすすめではない。

```ts
<NextLink href="/home" passHref>
  <Link href="/home">Link</Link>
</NextLink>
```

## `hrefPass` とは

[https://nextjs.org/docs/api-reference/next/link](https://nextjs.org/docs/api-reference/next/link)

> passHref - Forces Link to send the href property to its child. Defaults to false

Link のタグの子供の component に強制的に href を渡すしくみのようです。

ソースコードを見てみるとこの辺りで強制的に href を変えているのが見て取れます。

[https://github.com/vercel/next.js/blob/master/packages/next/client/link.tsx#L320](https://github.com/vercel/next.js/blob/master/packages/next/client/link.tsx#L320)

通常の a タグを利用する場合は意識しないので、Link タグの子供に別の component を配置する場合は注意が必要ですねー。
