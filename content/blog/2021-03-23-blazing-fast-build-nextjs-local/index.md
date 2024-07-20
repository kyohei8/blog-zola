+++
title="最速でNext.jsのローカル環境を作成する"
[taxonomies]
tags=["Next.js"]
+++

create-next-app とかよりもっとプレーンな Next.js の環境をサクッと作りたい場合。

ディレクトリを作成して、、

```bash
mkdir next-app && cd next-app
# もしくは take next-app
```

以下のコマンドを実行

```bash
npm init -y
npm install next react react-dom
npm install typescript @types/react @types/node -D
mkdir src
mkdir src/pages
mkdir src/components
touch .env.local
{
  echo "import { NextPage } from 'next';"
  echo "const Index: NextPage<{}> = () => <div>Index</div>;"
  echo "export default Index;"
} > src/pages/index.tsx
npm pkg set scripts.dev="next dev"
npm pkg set scripts.build="next build"
npm pkg set scripts.start="next start"

# npm v8 以下の場合
# npm set-script dev "next dev"
# npm set-script build "next build"
# npm set-script start "next start"
```

以上です。

あとは起動するだけ。起動時に tsconfig を自動で作成してくれます。

```bash
npm run dev
```

yarn 版

```bash
yarn init -y
yarn add next react react-dom
yarn add typescript @types/react @types/node -D
mkdir src
mkdir src/pages
mkdir src/components
touch .env.local
{
  echo "import * as React from 'react';"
  echo "import { NextPage } from 'next';"
  echo "const Index: NextPage<{}> = () => <div>Index</div>;"
  echo "export default Index;"
} > src/pages/index.tsx
npm pkg set scripts.dev="next dev"
npm pkg set scripts.build="next build"
npm pkg set scripts.start="next start"
```

```bash
yarn dev
```

2023/10 追記

App Router 版

```bash
npm init -y
npm install next@latest react@latest react-dom@latest
npm install typescript @types/react @types/node -D
mkdir src
mkdir src/app
mkdir src/components
touch .env.local
{
  echo "export default function RootLayout({children}: {children: React.ReactNode}) {"
  echo "return <html lang="en"><body>{children}</body></html>}"
} > src/app/layout.tsx
{
  echo "export default function Page() { return <h1>Hello, Next.js!</h1> }"
} > src/app/page.tsx
npm pkg set scripts.dev="next dev"
npm pkg set scripts.build="next build"
npm pkg set scripts.start="next start"
npm pkg set scripts.lint="next lint"
```

## options

### tailwindcss@3 を追加

[https://tailwindcss.com/docs/guides/nextjs](https://tailwindcss.com/docs/guides/nextjs)

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
# postcss.config.jsとtailwind.config.jsが作成される
```

tailwind.config.js の content に tsx ファイルの位置を指定する

```bash
module.exports = {
  content: ['./src/**/*.tsx'],
  ...
}
```

`global.css` を作成

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### tailwindcss@2.1(jit 版)を追加

[https://github.com/vercel/next.js/tree/canary/examples/with-tailwindcss](https://github.com/vercel/next.js/tree/canary/examples/with-tailwindcss)

※ 2021/04/25 時点のnext@10.1.3 の場合だと、postcss がコンフリクトする。その場合`--legacy-peer-deps` オプションを付け、古い依存関係を無視するようにする。

```bash
npm i autoprefixer postcss -D
npm i tailwindcss@2.1 -D
# tailwind.config.jsを生成
npx tailwindcss init
touch postcss.config.js
```

postcss.config.js を追加

```js
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {}
  }
};
```

tailwind の mode を `jit` に変更 ( tailwind.config.js を追加 )

purge に対象の JSX コンポーネントを指定する。

```js
module.exports = {
  mode: 'jit',
  purge: ['./pages/**/*.{js,ts,jsx,tsx}', './components/**/*.{js,ts,jsx,tsx}'],
  ...
}
```

`page/_app.tsx` に tailwindcss をインポート

```bash
import 'tailwindcss/tailwind.css';
```
