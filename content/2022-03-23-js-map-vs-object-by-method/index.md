+++
title="JavaScript Map vs Object（実装編）"
[taxonomies]
tags=["JavaScript", "Map", "Object"]
+++

JavaScript では単純な連想配列（ハッシュマップ）を実装する場合、Map または Object を利用することが多いと思う、使い方がよく似ているためどちらを選択していいか迷うときがある、今回はその違い、パフォーマンス、使い所をまとめてみた。

### そもそも...

JavaScript は元来(ES5 までは)、プリミティブな型以外の”データの集合”を表す場合、すべて Object となっており、Array や Function も実は Object である。

ES6 において、class や Set などデータの集合を表すための表現がいくつか追加され、その中の一つに Key-Value のデータを扱うための新しいアプローチとして Map も追加された。

とはいえ、JSON.parse のように、純粋な Object 型を使うケースはまだまだ存在し、どのようなタイミングで Map を利用するかが不明瞭だったので、それを少し調べてみた。

## Map VS Object

### Key

Map の一番の特徴は Key に String 以外の値を設定できるところかと思う。

```js
const m = new Map();
// こういったケースが可能
m.set(true, false);
m.set([0], '1');
m.set({}, '2');
m.set(() => {}, '3');
```

また、Object 型のインスタンスは Object の prototype を参照することになるので、valueOf や toString などのキーをデフォルトで持つことになる。また、最近は見なくなったけど、Object.prototype を拡張している場合そのプロパティも参照できてしまうことになる。（そのため、hasOwnProperty を挟むとか昔はよくあった）

```js
const x = {};
x['toString'] => Function

Object.prototype.y = 'yyy';
x.yyy => 'yyy'

for(const key in x){ console.log(key) } // yが取得できる
```

Map ではこのようなことが起きない。

### デフォルトの値

Map ではコンストラクタの引数に key-valeu 配列の配列を渡すことでデフォルトの値を定義することができる。

```js
const m = new Map([
  ['key', 1],
  ['key2', 2]
]);
m.get('key'); // 1
```

Object では宣言時のオブジェクトリテラル `{}` 内に値を定義する

```js
const o = { key: 1, key2: 2 };
o.key; // 1
```

### 値の登録

Map では `set` メソッドを利用して値を登録する。key が重複した場合は当然値が上書きされる。

```js
const m = new Map();
m.set('key', 1);
m.set('key', 2); // keyの値を上書き
```

Object では `.` で key を直接指定、またはハイフンを含むキーや全角文字の場合クォートを付けた文字列リテラルまたは文字列の変数を利用して、設定する方法がある。

```js
const o = {};
o.key = 1;
o['key-2'] = 2;
```

### 値の取得

Map における値の取得は `get` メソッドを利用する。key が存在しない場合は `undefinde` が返る。

```js
const m = new Map();
m.set('key', 1);
m.get('key'); // 1
m.get('key123'); // undefined
```

Object は設定時と同様で `.` で key を直接指定、または `[]` 内に文字列を入れて呼び出す方法がある。キーが存在しない場合は `undefined` が返る。

```js
const o = {};
o.key = 1;
o.key; // 1
o.key123; // undefined
```

### 値の削除

Map では `delete` メソッドを利用して指定のキー及び値を削除することができる。削除が成功すると、 `true` を返し、key が存在しない場合は、 `false` を返す。

```js
const m = new Map();
m.set('key', 1);
m.delete('key'); // true
m.delete('key123'); // false;
```

また、map 内のすべての値を削除する `clear` メソッドも用意されている。

```js
m.clear();
```

Object では、 `delete` 演算子を用いてキー及び値を削除する。キーが自分自身の non-configurable のプロパティであった場合、 strict モードでなければ false を返し、それ以外は true を返す。つまりキーが存在しない場合でも true を返す。

[https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Cant_delete](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Cant_delete)

```js
const o = { key: 1 };
delete o.key; // true
delete o.y; // true
```

### キーの存在確認

Map でキーの存在を確認する場合は `has` メソッドを利用します。キーが存在する場合は true、存在しない場合は false を返す。

```js
const m = new Map();
m.set('key', 1);
m.has('key'); // true
m.has('key123'); // false
```

Object では `hasOwnProperty` メソッドで自身のオブジェクトにあるキーかをチェックすることができる。

```js
const o = { key: 1 };
o.hasOwnProperty('key'); // true
// もしくは
Object.prototype.hasOwnProperty(o, 'key'); // true
```

<aside>
💡 自分の所感ではget等で呼び出して、 `undefined` かどうかでチェックする構文をよく利用する。この場合のコーナーケースはキーはあるけど値が `undefined` である場合と、そもそもキーがない場合とで見分けがつかないことである。
とはいえ、できれば正しく判定メソッドを利用すべきである。

</aside>

### 要素数の確認 / 空チェック

Map では `size` メソッドが用意されており、簡単に要素数を取得できる。値が 0 であるかを判定する事によって empty かのチェックもできる。

```js
const m = new Map([['key', 1]]);
m.size; // 1
```

Object では `Object.entries` 、 `Object.keys` 、 `Object.values` にて配列に変換し、その要素数を取得するというのが一般的なやり方である。一度配列に変換するので、コストがかかるという印象。

余談だが、 `Object.keys` よりも `Object.values`の方が Chrome,Firefox では[パフォーマンスが良い](https://perf.link/#eyJpZCI6IjVjbXkxYmMwYXI0IiwidGl0bGUiOiJ0cmF2ZXJzZSIsImJlZm9yZSI6ImNvbnN0IG4gPSAxMDAwO1xuY29uc3QgZGF0YSA9IFsuLi5BcnJheShuKS5rZXlzKCldO1xuY29uc3QgbSA9IG5ldyBNYXAoKTtcbmRhdGEuZm9yRWFjaCgoZCkgPT4gbS5zZXQoZCwgZCkpO1xuY29uc3QgbyA9IHt9O1xuZGF0YS5mb3JFYWNoKChkKSA9PiBvW2RdID0gZCk7IiwidGVzdHMiOlt7Im5hbWUiOiJUZXN0IENhc2UiLCJjb2RlIjoiT2JqZWN0LnZhbHVlcyhvKS5sZW5ndGgiLCJydW5zIjpbODAwMCwyMDAwMCw2MDAwLDEwMjAwMCw5NjAwMCwxNjIwMDAsNDcwMDAsMTIxMDAwLDY1MDAwLDIyOTAwMCwxNzUwMDAsMTI2MDAwLDIxMjAwMCw0MjAwMCw2MDAwLDExNjAwMCwxODAwMDAsOTEwMDAsMTkwMDAsMjMwMDAsMTQ2MDAwLDEzNTAwMCwxNDgwMDAsMjgwMDAsMjE2MDAwLDIxMDAwLDE5MDAwMCwyNTkwMDAsMjIzMDAwLDEwMDAwLDU0MDAwLDkwMDAsNTIwMDAsMzIwMDAsMTAwMDAsMTI1MDAwLDE4NjAwMCw0MzAwMCwxOTYwMDAsODAwMDAsMTg0MDAwLDUxMDAwLDIxMTAwMCwzMDAwLDk4MDAwLDIyMDAwLDU4MDAwLDEzOTAwMCw5MDAwMCwyMjgwMDAsNzIwMDAsMTA1MDAwLDU3MDAwLDU4MDAwLDE2MTAwMCwzMjUwMDAsMjM3MDAwLDYyMDAwLDU1MDAwLDE5NDAwMCwzMjAwMCwxNzQwMDAsMTE5MDAwLDE4NzAwMCwyMDgwMDAsOTgwMDAsMjI4MDAwLDM1MDAwLDIyNzAwMCwxNDYwMDAsMTI3MDAwLDcyMDAwLDIwNzAwMCwzMTAwMCwyNjMwMDAsMTgzMDAwLDE1MTAwMCwxNzAwMCwxMDAwMCwxMTIwMDAsNzYwMDAsMTAxMDAwLDQzMDAwLDI3MDAwLDEwNDAwMCw5MDAwLDUyMDAwLDM5MDAwLDQ0MDAwLDcwMDAwLDE1NjAwMCwyMDUwMDAsNDAwMDAsODQwMDAsMjI2MDAwLDkwMDAsMTczMDAwLDk4MDAwLDE3NDAwMCwxMjMwMDBdLCJvcHMiOjEwODk5MH0seyJuYW1lIjoiVGVzdCBDYXNlIiwiY29kZSI6Ik9iamVjdC5lbnRyaWVzKG8pLmxlbmd0aCIsInJ1bnMiOls3MDAwLDEwMDAwLDMwMDAsMTAwMDAsMTcwMDAsOTAwMCw2MDAwLDgwMDAsMTAwMDAsNjAwMCwxODAwMCwyMDAwLDEzMDAwLDEwMDAsMTAwMCwxMDAwMCwxNzAwMCw5MDAwLDExMDAwLDMwMDAsNTAwMCwxMTAwMCw4MDAwLDE3MDAwLDE5MDAwLDEwMDAwLDEwMDAsMjEwMDAsMjMwMDAsNTAwMCwzMDAwLDIzMDAwLDQwMDAsMzAwMCwxMDAwLDgwMDAsMTkwMDAsMzAwMCwxMzAwMCw4MDAwLDEyMDAwLDMwMDAsMTcwMDAsMjAwMCwxMTAwMCwxMTAwMCwxNzAwMCwzMDAwLDYwMDAsMTEwMDAsNjAwMCwxMDAwMCwzMDAwLDMwMDAsMTEwMDAsMjUwMDAsMjIwMDAsNTAwMCw4MDAwLDEyMDAwLDgwMDAsMzAwMCw4MDAwLDEwMDAwLDE4MDAwLDQwMDAsMzAwMCwzMDAwLDMwMDAsNzAwMCwxNzAwMCw5MDAwLDE5MDAwLDgwMDAsMTQwMDAsMTcwMDAsNTAwMCwxNTAwMCwxMDAwLDE3MDAwLDMwMDAsMzAwMCw0MDAwLDIwMDAsODAwMCw1MDAwLDEyMDAwLDMwMDAsMjkwMDAsODAwMCw5MDAwLDExMDAwLDQwMDAsOTAwMCwxNDAwMCwxNzAwMCwxMDAwMCwyMDAwLDkwMDAsOTAwMF0sIm9wcyI6OTM0MH0seyJuYW1lIjoiVGVzdCBDYXNlIiwiY29kZSI6Ik9iamVjdC5rZXlzKG8pLmxlbmd0aCIsInJ1bnMiOlsyMzAwMCw4MDAwLDYwMDAsMzAwMDAsNTAwMDAsMzQwMDAsODAwMCwyMTAwMCwyMjAwMCw2MDAwLDQ0MDAwLDQxMDAwLDU2MDAwLDgwMDAsNzAwMCw0NTAwMCw0MDAwMCwyNzAwMCw4MDAwLDQ3MDAwLDgwMDAsMTUwMDAsMTgwMDAsMjAwMCw1ODAwMCw4MDAwLDQ3MDAwLDYzMDAwLDUwMDAwLDEwMDAwLDUwMDAsMzkwMDAsMTMwMDAsNDgwMDAsODAwMCw5MDAwLDMxMDAwLDkwMDAsMzAwMDAsODAwMCwyNDAwMCw4MDAwLDE5MDAwLDYwMDAwLDMxMDAwLDI4MDAwLDkwMDAsMTAwMDAsODAwMCw2NDAwMCwzMDAwMCw0NDAwMCw3MDAwLDQwMDAsMzUwMDAsOTAwMDAsNjMwMDAsMzAwMDAsODAwMCw1NjAwMCwxMDAwLDgwMDAsMTAwMDAsMjQwMDAsMzUwMDAsMjAwMCw0NTAwMCwxMDAwLDM3MDAwLDIyMDAwLDUyMDAwLDQ0MDAwLDQ3MDAwLDE2MDAwLDI2MDAwLDMzMDAwLDE1MDAwLDQ4MDAwLDUwMDAsMjEwMDAsMzAwMDAsMTQwMDAsODAwMCw4MDAwLDgwMDAsOTAwMCw1NTAwMCw4MDAwLDEwMTAwMCwxMzAwMCwzNDAwMCw0NjAwMCw4MDAwLDUyMDAwLDYzMDAwLDgwMDAsMjUwMDAsOTAwMCw1NTAwMCwzMDAwXSwib3BzIjoyNjg3MH1dLCJ1cGRhdGVkIjoiMjAyMi0wMy0yNVQwMzo1MjoxOC44NTdaIn0%3D)。

```js
const o = { key: 1 };
Object.values(o).length; // 1
```

### 走査

Map, Object ともに走査の方法はいくつか存在する。

Map はイテレータブルなオブジェクトなので、 `for .. of` が利用できる。この場合、[ key, value ]のタプルような形で値を取得される。

```js
const m = new Map([
  ['key1', 1],
  ['key2', 2]
]);

for (const p of m) {
  console.log('key', p[0], 'value', p[1]);
}
// こちらと同等の処理である
for (const p of map.entiers()) {
  console.log('key', p[0], 'value', p[1]);
}
```

`keys` , `values` メソッドでキーのみ、値のみのイテレータブルなオブジェクトを取得することができる。

map オブジェクトに実装されている `forEach` を利用しても同様に走査ができる。（Array の forEach とは引数が異なるので注意）

```js
m.forEach((v, k) => console.log(k, v));
```

イテレータブルなオブジェクトは `Array.from` または `[...x]` を利用して配列に変換できるため、従来の配列のメソッドを利用したい場合はこちらのパターンを利用する。

```js
[...m].forEach(([k, v]) => console.log(k, v));
// または [...m.entiers()],[...m.keys()],[...m.values()]
```

Object の場合は `for..in` を利用して走査ができる。

ただし、この場合、前述の通り Object.prototype に拡張されているプロパティまで参照されてしまう。なので、hasOwnProperty 等でチェックする必要がある。

```js
const o = {key1: 1, key2: 2};
Object.prototype.x = 3
for (const p in o) {
  console.log('key', p, 'value', o[p]);
  // key key1 value 1
  // key key2 value 2
  // key x value 3  <- !!
}

for (const p in o) {
  if(o.hasOwnProperty(p){
    console.log('key', p, 'value', o[p]);
  }
  // key key1 value 1
  // key key2 value 2
 }
```

結論から言うと、for..in を使うケースは今後ほぼないと思います。
代わりに `Object.entries()` `Object.key()` `Object.values()` のいずれかを使い、配列に変換し、Array のメソッドもしくは for..of を利用して走査するのが一般的です。

```js
Object.entries(o).forEach(([k, v]) => console.log(k, v));

// または

for (const p of Object.entries(o)) {
  console.log(p[0], p[1]);
}
```

### Map のオブジェクト、配列への変換

ES2019 から加わった `Object.fromEntries` を使うことにより、オブジェクトへの変換が可能、

また、 `Array.from` で、配列に変換できます。

```js
const m = new Map([
  [1, 2],
  [3, true]
]);

Object.fromEntries(m);
// {1: 2, 3: true}

Array.from(m);
// [[1, 2], [3, true]]
```

### JSON へのシリアライズ

Map の場合、キーに文字列以外を指定できるため、原則的には直接変換できないものと思われる。

上記の `Object.fromEntries` を利用し、オブジェクトに変換し、それをシリアライズすることは可能。ただし、キーに文字列以外が指定されていると意図しない結果になるので注意。

```js
const m = new Map([[1, 1]]);
const o = { obj: '123' };
m.set(o, 100);
JSON.stringify(Object.fromEntries(m));
// '{"1":1,"[object Object]":100}'  🤔
```

また、配列に変換しそれをシリアライズすることが可能。こちらもキーに関数やクラスのインスタンスなど設定すると意図しないものとなるので注意。

```js
const m = new Map([[1, 1]]);
const o = { obj: '123' };
m.set(o, 100);

JSON.stringify([...m]); // もしくは Array.from(m.entries())など
// => [[1,1],[{"obj":"123"},100]]

// キーに関数やクラスのインスタンスなど設定すると意図しないものとなるので注意
const m2 = new Map();
const f = (x) => x + x;
class A {
  get() {
    return 'get';
  }
}
const aa = new A();
m2.set(f, 'func');
m2.set(aa, aa);

JSON.stringify([...m2]);
// => [[null, 'func'],[{}, {}]] 🤔
```

### まとめ

|                      | Map                 | Object                |
| -------------------- | ------------------- | --------------------- |
| constructor          | const m = new Map() | const o = {}          |
| 値の登録             | m.set(key, value)   | o.key = value         |
| o[key] = value       |
| 値の取得             | m.get(key)          | o.key                 |
| o[key]               |
| 値の削除             | m.delete(key)       |
| m.clear()            | delete o.key        |
| delete o[key]        |
| プロパティの存在確認 | m.has(key)          | o.hasOwnProperty(key) |

| プロパティ数の確認
空かどうかのチェック | m.size
m.size === 0 | Object.values(o).length
Object.values(o).length === 0 |
| 走査 | for(const p of m){...}
m.forEach((v, k) => {...});
[...m].forEach((p) => {...}) | Object.entries(o).forEach((p) => {...})
for(const p of Object.entries(o)){...} |
| オブジェクト化、配列化 | Object.fromEntries(m)
Array.from(m) | Object.entries(o) |

同等のことができとわかったので、次回はパフォーマンスを見てみたいと思います。

次回 ↓

[JavaScript Map vs Object （パフォーマンス編） | kyohei's blog](https://1k6a.com/blog/js-map-vs-object-performance)
