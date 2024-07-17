+++
title="TypeScriptのインデックスシグネチャについて"
[taxonomies]
tags=["TS7017", "TypeScript"]
+++

JavaScript から移行して TypeScript を書いていると、オブジェクトの扱いで以下のようなエラーが発生することがある。

```
Element implicitly has an 'any' type because type '{}' has no index signature.
（型 '{}' にはインデックス シグネチャがないため、要素は暗黙的に 'any' 型になります。）
```

具体的には以下のようなコードで発生する

```ts
const obj: {} = {
  a: 123
};

obj['b'] = 1;
```

### インデックス シグネチャとは

JavaScript のオブジェクトは元来、キーとなる値は文字列以外に数値や Symbol 型などを指定できる。

e.g.

```js
var o = {};

o['a'] = 'aa';
o[123] = 'bb';

var s = Symbol('symbol1');
o[s] = 'cc';

console.log(o);
// > {123: "bb", a: "aa", Symbol(symbol1): "cc"}
```

TypeScript ではこのいずれかのキーを指定できるという状況を避けるようにするため、キーの型が何であるかを指定する必要がある。

```ts
const obj: {
  [key: String]: number;
} = {
  a: 123
};

obj['b'] = 1;
```

この `[key: String]: number`の部分が**インデックスシグネチャ**になる。
`key`の部分は任意なので別のものでもよいです。

多くの場合、key の方は`string`になるが、特定の値のみをキーに指定することもできる。

```ts
const obj: { [key in 'a' | 'b']?: string } = {
  a: 123
};

obj['b'] = 1; // OK
obj['c'] = 2; // Error
```
