+++
title="TypeScriptでimportを絶対パスの参照にする"
[taxonomies]
tags=["Next.js", "TypeScript"]
+++

TypeScript で ES Module を import を絶対パスで参照するやり方です。

絶対パスで import 文を書くと以下のようになります。

```tsx
import { Heading } from '../../../components/common/Typography';
import { UserTypes } from '../../../constants';
```

↓

```tsx
import { Heading } from '@/components/common/Typography';
import { UserTypes } from '@/constants';
```

## そもそもなぜ絶対パスなのか？

メリットデメリットをまとめてみました。個人的にはおすすめではありますが、マストではないかもです。

### メリット

- import 構文における、ドット地獄からの開放
  - （ルートからのパスを記述するので）import するファイルがどの位置にあるかわかりやすい
- ファイルを移動させても、import 文を変更する必要はない
- import 対象ファイルがルートから近くなるにつれでパスの長さは短くなる

### デメリット

- 通常、エディタの補完は相対パスなので、エディタ側でパスの補完をする場合はその設定が必要
  - VSCode の場合は `"typescript.preferences.importModuleSpecifier"` で設定可
- ファイル名、ディレクトリ名を変更した際、影響するファイルが増える
- import 対象のファイルが近くにある場合は、相対パスで表記するより長くなる場合がある

---

## TypeScript での設定の仕方

Next.js(with TypeScript)で設定する場合、tsconfig の compilerOptions に import するルート位置とパスの略称を追加します。

paths には接頭辞となるシンボルを設定しています。今回は `@/` を指定してますが、別のシンボルでも構いません。

逆にシンボルがない場合はライブラリや Node.js のモジュールと区別がつかなくなるため、利用できません。（webpack を利用する場合は優先順位があってシンボルなしでもできたような気もしますが・・）

```tsx
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },
    ...
  }
}
```

TypeScript での設定は以上になります。jest や storybook を利用している場合は以下の追加の設定が必要です。

### jest の設定

jest を実行する際にモジュールの＠部分を変換します。

```tsx
// jest.config.ts

export default {
  ...
  moduleNameMapper: {
    ...
    // importの@を解決
    '@/(.*)': '<rootDir>/src/$1'
  },
}
```

### storybook の設定

storybook も同様にパスを解決するように設定します。

参考: [https://github.com/storybookjs/storybook/issues/3916#issuecomment-871283551](https://github.com/storybookjs/storybook/issues/3916#issuecomment-871283551)

```tsx
// .storybook/main.js

const path = require('path');

module.exports = {
  ...
  webpackFinal: async config => {
    // importの@を解決
    config.resolve.alias = {
      ...config.resolve.alias,
      '@': path.resolve(__dirname, '../src')
    };

    return config;
  }
}
```
