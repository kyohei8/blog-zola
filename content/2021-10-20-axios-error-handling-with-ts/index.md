+++
title="axiosのエラーハンドリング"
[taxonomies]
tags=["TypeScript", "axios"]
+++

## TL; DR:

```tsx
import axios, { AxiosError } from 'axios';

try {
  await axios.get('...');
} catch (e) {
  if ((e as AxiosError).isAxiosError && e.response) {
    // axiosのエラー
    throw new Error(e.response.data);
  } else {
    // 通常のエラー
    throw new Error(e);
  }
}
```

## 説明

axios でエラー(レスポンスが 5XX,4XX)が発生した場合、通常の Error メッセージだと `Request failed with status code 500` とかになり、axios のライブラリ上で定義されているエラーメッセージが返る。

[https://github.com/axios/axios/blob/v0.23.0/lib/core/settle.js#L18](https://github.com/axios/axios/blob/v0.23.0/lib/core/settle.js#L18)

```tsx
try {
  await axios.get('...');
} catch (e) {
  console.log(e.message); // Request failed with status code 500
}
```

xhr のレスポンスの body にエラーメッセージあり、これを取得したい場合、Error オブジェクトは `AxiosError` のインスタンスとなり、その中に body のデータがあります。

手順としては

1. Error のインスタンスが AxiosError かを判定します。
   1. AxiosError インスタンスには `isAxiosError` というプロパティがあり、それで AxiosError かそれ以外の Error かを判定できる。（または `axios.isAxiosError(e)` でも同様の判定ができる）
2. AxiosError だった場合、`AxiosError.response.data` にそのデータがあるので取得する
   1. 同様に http status や header も取得することができる

```tsx
try {
  await axios.get('...');
} catch (e) {
  if ((e as AxiosError).isAxiosError && e.response) {
    console.log(e.response.data); // error message from reponse body
  }
}
```

ベストプラクティスかどうかはわからないですが、ある程度はこれで判定できると思います。
