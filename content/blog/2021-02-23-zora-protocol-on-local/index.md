+++
title="Zora Coreをローカルで動かす"
[taxonomies]
tags=["Ethereum", "zora"]
+++

[zora core](https://github.com/ourzora/core) をローカルのチェーン(Ganashe)で動かす方法です。

## clone, install

```bash
git clone git@github.com:ourzora/core.git
cd core
yarn
```

## env ファイルの作成

ルートディレクトリに`.env.local`を作成。RPC_ENDPOINT と PRIVATE_KEY を設定する。
RPC_ENDPOINT は Ganache のエンドポイント、PRIVATE_KEY はデプロイするアカウントの秘密鍵

```
RPC_ENDPOINT=http://127.0.0.1:7545
PRIVATE_KEY=b3fc...
```

## 空の addresses ファイルを作成

ChainId のファイルを作成し中身は空（`{}`）にします。
（Ganache はデフォルトの ChainId は 1337）

```bash
echo '{}' > addresses/1337.json
```

## デプロイを実行

```bash
yarn build
yarn deploy --chainId 1337
```

デプロイが完了すると、address/1337.json が更新されます

```
{
  "market": "0x1ec...",
  "media": "0x6Be..."
}
```

デプロイしたコントラクトのアドレスを、zdk のコンストラクタに指定する。

```ts
const zora = new Zora(
  signer,
  chainId, // この場合chainIdは無視される
  '0x6Be...',
  '0x1ec...'
);
```
