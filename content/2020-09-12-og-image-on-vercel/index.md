+++
title="og画像をvercel上で生成してみる"
[taxonomies]
tags=["Lambda", "TypeScript", "puppeteer", "vercel"]
+++

ブログの記事などで OG 画像を動的に作りたい場合、サーバーサイドを持つサービスなら、 [ImageMagick](https://imagemagick.org/script/index.php) や [node-canvas](https://github.com/Automattic/node-canvas) を使い構築することができますが、サーバーサイドを持たない静的なサイトの場合、 vercel 等の Serverless Functions を使い、画像を生成するマイクロサービスを作成し運用することができます。

## og-image について

[og-image](https://github.com/vercel/og-image) は vercel 社が作成・メンテナンスをしている OG 画像を動的に生成する方法で、仕組みとしては入力されたデータで html を出力し、その後ヘッドレス Chrome である [puppeteer](https://github.com/puppeteer/puppeteer) を利用しキャプチャを生成するというもので、html で生成することによって座標の計算等ややこしい部分を柔軟に対応できる。

## 試してみる

og 画像を生成する仕組みを作り、deploy する流れを簡単に作成してみます。

まずは package.json と、TypeScript を利用するため、以下のコマンドで生成・インストールします。

```bash
npm init
npm i typescript@3.9 -D
```

次に、vercel.json を追加し、以下のように設定します。

api のメモリ 1024MB は[vercel の無料枠である Hobby のギリギリのライン](https://vercel.com/docs/platform/limits#serverless-function-memory)です。

rewrites はすべてのリクエストを api/index に向けます。これにより URL のドメイン以下のパスにテキストを入れても index に到達するようになります。

```json
{
  "regions": ["hnd"],
  "functions": {
    "api/**": {
      "memory": 1024
    }
  },
  "rewrites": [{ "source": "/(.+)", "destination": "/api" }]
}
```

次に api/tsconfig.json を作成します。お好みで設定してください。

```json
{
  "compilerOptions": {
    "outDir": "dist",
    "module": "commonjs",
    "target": "esnext",
    "moduleResolution": "node",
    "jsx": "react",
    "sourceMap": true,
    "strict": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitReturns": true,
    "noEmitOnError": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "removeComments": true,
    "preserveConstEnums": true,
    "esModuleInterop": true
  },
  "include": ["./"]
}
```

Serverless Function(api/index.tsx)を作成し、ひとまずは html を返す設定にします。

```ts
import { IncomingMessage, ServerResponse } from 'http';

export default async function handler(_req: IncomingMessage, res: ServerResponse) {
  try {
    res.setHeader('Content-Type', 'text/html');
    res.end('<h1>Hello</h1>');
  } catch (e) {
    res.statusCode = 500;
    res.setHeader('Content-Type', 'text/html');
    res.end('<h1>Internal Error</h1><p>Sorry, there was a problem</p>');
    console.error(e);
  }
}
```

次に vercel の設定を行い、設定をします。

```
$ vercel
Vercel CLI 19.1.2
? Set up and deploy “~/workmy/og-image-blog”? [Y/n] y
? Which scope do you want to deploy to?
? Link to existing project? [y/N] n
? What’s your project’s name? og-image-blog
? In which directory is your code located? ./
No framework detected. Default Project Settings:
- Build Command: `npm run vercel-build` or `npm run build`
- Output Directory: `public` if it exists, or `.`
- Development Command: None
? Want to override the settings? [y/N] n
...
```

vercel に一度デプロイされますが、一旦それは無視してローカルで作業します

```
$ vercel dev
```

localhost:3000 にアクセスするとディレクトリのリストが見えますが、一旦そちらは無視し、`localhost:3000/demo` にアクセスします。

vercel.json の rewrite でインデックスページ以外のパスはすべて api/index に転送されるように設定しているので、アクセスすると <h1>でくくられた Hello の文字が表示されます。

### URL に渡された文字列を取得し表示

`http://localhost:3000/foo-bar` のように URL で渡された文字列をパースし、画面に表示します。

```ts
import { IncomingMessage, ServerResponse } from 'http';
import { parse } from 'url';

export default async function handler(req: IncomingMessage, res: ServerResponse) {
  try {
    res.setHeader('Content-Type', 'text/html');
    const { pathname } = parse(req.url || '/', true);
    const path = (pathname || '/').slice(1);
    res.end(`<h1>${path}</h1>`);
  } catch (e) {
    //...
  }
}
```

## puppeteer を起動し、画像を取得

puppeteer をインストールします。

```
npm i puppeteer-core -S
npm i @types/puppeteer @types/puppeteer-core -D
```

api/index.ts を以下のように修正します

exePath にはローカルにある chrome のアプリケーションへのパスを指定します。windows、mac、linux で変わるのでそれぞれ指定します。

```ts
import { IncomingMessage, ServerResponse } from 'http';
import { launch, Page } from 'puppeteer-core';
import { parse } from 'url';

const exePath =
  process.platform === 'win32'
    ? 'C:\\\\Program Files (x86)\\\\Google\\\\Chrome\\\\Application\\\\chrome.exe'
    : process.platform === 'linux'
    ? '/usr/bin/google-chrome'
    : '/Applications/Google Chrome.app/Contents/MacOS/Google Chrome';

export default async function handler(req: IncomingMessage, res: ServerResponse) {
  try {
    const { pathname } = parse(req.url || '/', true);
    const path = (pathname || '/').slice(1);

    // ブラウザを起動
    const browser = await launch({
      args: [],
      executablePath: exePath,
      headless: true
    });
    const page = await browser.newPage();
    await page.setViewport({ width: 2048, height: 1170 });
    await page.setContent(`<h1>${path}</h1>`);
    const file = await page.screenshot({ type: 'png' });

    res.setHeader('Content-Type', `image/png`);
    res.end(file);
  } catch (e) {
    // ..
  }
}
```

これで [`localhost:3000/test`](http://localhost:3000/test) 等にアクセスするとパスで指定した文字列が画像で表示されます。

## デプロイ

vercel 上で動かすためにはサーバレス関数上で起動する Chrome を用意しないといけないため、
[chrome-aws-lambda](https://github.com/alixaxel/chrome-aws-lambda)を使用します。このパッケージは Chrome のバイナリを保有し、AWS lambda や Google Cloud Functions で puppetter を利用できるようにします。vercel の Serverless Function は lambda で動作しているためこのライブラリが利用できます。

```
npm i chrome-aws-lambda -S
```

import し、puppeteer の設定に追加します。

```ts
import chrome from 'chrome-aws-lambda';
import { IncomingMessage, ServerResponse } from 'http';
import { launch } from 'puppeteer-core';
import { parse } from 'url';

let page: any;

const exePath = //..

export default async function handler(
  req: IncomingMessage,
  res: ServerResponse
) {
  try {
    const { pathname } = parse(req.url || "/", true);
    const path = (pathname || "/").slice(1);

    // AWS_REGIONが存在するかどうかでローカル・vercel環境かを判定
    const isDev = !process.env.AWS_REGION;
    // ブラウザを起動
    const browser = await launch(
      isDev
        ? {
            args: [],
            executablePath: exePath,
            headless: true,
          }
        : {
            args: chrome.args,
            executablePath: await chrome.executablePath,
            headless: chrome.headless,
          }
    );
    page = await browser.newPage();
    await page.setViewport({ width: 2048, height: 1170 });
    await page.setContent(`<h1>${path}</h1>`);
    const file = await page.screenshot({ type: "png" });

    res.setHeader("Content-Type", `image/png`);
    // og画像はそう変わることがないため、キャッシュを長めに設定します
    res.setHeader(
      "Cache-Control",
      `public, immutable, no-transform, s-maxage=31536000, max-age=31536000`
    );
    res.end(file);
  } catch (e) {
    // ...
  }
}
```

vercel にデプロイします。

```ts
vercel --prod
```

デプロイが成功すると、https://[yourdomain].vercel.app/hello で画像が表示されるようになりました。

![og%E7%94%BB%E5%83%8F%E3%82%92vercel%E4%B8%8A%E3%81%A6%E3%82%99%E7%94%9F%E6%88%90%E3%81%97%E3%81%A6%E3%81%BF%E3%82%8B%205d6b254fb4b442d292e131933c6839fc/hello!.png](og%E7%94%BB%E5%83%8F%E3%82%92vercel%E4%B8%8A%E3%81%A6%E3%82%99%E7%94%9F%E6%88%90%E3%81%97%E3%81%A6%E3%81%BF%E3%82%8B%205d6b254fb4b442d292e131933c6839fc/hello!.png)

（↑html に見えますが画像です。。😅）

## まとめ

このように puppetter を介して画像を生成するマイクロサービスが作成できました。
あとは表示する html を装飾することで（CSS で表現できるものや web font を利用したり等で）好みの画像を生成することができます。

### 問題点

- vercel の hobby(フリープラン)の最大メモリが 1024MB となり、処理にもよりますが若干処理が重い（遅い）です。遅いことによって twitter などの表示では画像が表示されないことがあります。。なので、production で検討するなら有償のプランを検討したほうがいいですね。
