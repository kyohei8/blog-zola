+++
title="async-semaについて"
[taxonomies]
tags=["Async", "JavaScript"]
+++

Java や C++などではマルチスレッドプログラミングではセマフォを使った実装はよくありますが、JavaScript では以下のようなケースでセマフォを利用したくなることがあります

- XHR で一度にデータを沢山読み込みたいが呼び出し先に API のリクエスト制限があったり、サーバに負荷をかけたくない場合に一定の数だけリクエストする場合
- （node.js で)ファイルや DB のデータに非同期でアクセスする際に pause/resume を利用したい場合
- js はマルチスレッド言語ではないが、webworker を利用して処理をしたい場合

[async-sema](https://github.com/vercel/async-sema) は JavaScript の async-await でセマフォを実装したライブラリです。

## Usage

以下の例は 20 個分の要素（データ）を最大 3 つずつ同時に処理する例です

```js
const { Sema } = require('async-sema');

const wait = () => new Promise((resolve) => setTimeout(resolve, 1000));

async function main() {
  const items = [];

  for (let i = 0; i < 20; i++) items.push(i + 1);

  const s = new Sema(3);
  await Promise.all(
    items.map(async (elem) => {
      await s.acquire();
      console.log(elem, s.nrWaiting());
      await wait();
      s.release();
    })
  );
  console.log('done');
}

main();
```

要点は３つで

- `new Sema()` でセマフォを定義。引数は同時に処理する数を指定する
- `s.acquire()` でセマフォからトークンを取得。利用可能なリソースを減らす(デクリメント(P 操作))する。
- `s.release()` でセマフォを開放し、利用可能なリソースを増やす(インクリメント(V 操作))

全体を Promise.all でラップします。

`s.nrWaiting()` で残りの処理数を出力します。

## その他のオプション

コンストラクタで用意されている引数のオプションには以下の物があります

- `initFn` セマフォのトークンを管理する場合に利用します。デフォルトではトークは'1'(`() ⇒ '1'`)に設定されています。
- `pauseFn` リクエストを一時的に中断する関数を指定することができます
- `resumeFn` リクエストを再開する関数を指定することができます
- `capacity` セマフォに割り当てる処理のリストがいくつあるかを指定します。通常はパフォーマンス向上のため利用される

## メソッド

- `drain()` セマフォをドレインし、初期化されたすべてのトークを配列で返します。プロセスを終了する前など、保留中の非同期タスクがないかを確認する事ができます。
- `tryAcquire()` トークンが利用可能かを確認する、利用できない場合は undefined が返る。
