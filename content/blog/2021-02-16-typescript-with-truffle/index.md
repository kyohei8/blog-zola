+++
title="TypeScriptでTruffleの環境を作成する"
[taxonomies]
tags=["Ethereum", "Truffle", "TypeScript"]
+++

ちょっと久しぶりに Ethereum(Dapp)の勉強をしているので、いろいろナレッジを書き残していきたい。

---

こちらの TypeChain を参考にしています 👀

[https://github.com/ethereum-ts/TypeChain](https://github.com/ethereum-ts/TypeChain)

前提条件

```markdown
Truffle v5.1.66 (core: 5.1.66)
Solidity - 0.6.2 (solc-js)
Node v14.15.3
Web3.js v1.2.9
```

Node.js は v14 だと、truffle unbox react が動かなかったりするので v12 でもいいと思う。

ディレクトリを作成し、各種ファイルをインストール

```markdown
$ mkdir truffle-ts-demo
$ cd truffle-ts-demo

# package.json を作成

$ npm init --y

# 必要なパッケージをインストール

$ npm install truffle typechain @typechain/truffle-v5 typescript -D
$ npm install @types/bn.js @types/mocha @types/chai @types/web3 -D
```

## npm script の作成

package.json の scripts 部分を以下のように変更する

```json
"scripts": {
  "generate-types": "typechain --target=truffle-v5 'build/contracts/*.json'",
  "postinstall": "truffle compile && npm run generate-types",
  "build-migration-files": "tsc -p ./tsconfig.migrate.json --outDir ./migrations",
  "migrate": "npm run build-migration-files && truffle migrate",
  "test": "npm run postinstall && truffle test"
},
```

### generate-types

build/contract/\*.json から types（typescript の型ファイル）を生成。test ファイル等の型依存をこのファイルで管理する。

### postinstall

solidity のコンパイルを行い、型ファイルを生成する。sol ファイルの型を変更した場合は都度実行する必要がある。

### migrate

マイグレーションを実行。migration ファイルも TypeScript で記述されているため、一度 js ファイルを作成し、そのファイルを元にマイグレーションを行う。

### test

ビルドを実行し、テストを実行します。

## tsconfig.json を作成

tsconfig.json とマイグレーション用の tsconfig.migrate.json を作成する。

```markdown
$ touch tsconfig.json tsconfig.migrate.json
```

tsconfig.json は以下のようにする。お好みで変えてもよい。

```json
{
  "compilerOptions": {
    "target": "es2018",
    "module": "commonjs",
    "lib": ["ES2018", "DOM"],
    "sourceMap": true,
    "strict": true,
    "moduleResolution": "node",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}
```

tsconfig.migrate.json はマイグレーションファイルのトランスパイルに利用します。基本的には同じで対象のファイルの対象が異なることになります。

```json
{
  "extends": "./tsconfig.json",
  "include": ["./migrations-ts/*.ts", "./types/**/*.ts"]
}
```

## `truffle init`のような雛形を作成する。

```json
# まずディレクトリを作成します
$ mkdir contracts migrations-ts test
# truffleの設定ファイルを作成します。
$ touch truffle-config.js
```

truffle-config.js はとりあえず最低限の設定をしておきます

```js
module.exports = {
  networks: {
    development: {
      host: '127.0.0.1',
      port: 8545,
      network_id: '*'
    }
  },

  compilers: {
    solc: {
      version: '0.6.2'
    }
  }
};
```

## ソースファイルを作成し、マイグレーションを行う

Migrations.sol とそのマイグレーションファイルを作成します。

```sh
$ touch contracts/Migrations.sol
$ touch migrations-ts/1_initail_migration.ts
```

Migrations.sol は truffle init と同じ内容を記述します。1_initial_migration.ts は TypeScript で以下の内容を記述します。

```ts
const Migrations = artifacts.require('Migrations');

module.exports = function (deployer) {
  deployer.deploy(Migrations);
} as Truffle.Migration;

export {};
```

基本的には js と同じですが、`as Truffle.Migration` で型の担保を行っています。また、最後の行に`export {}`を付ける必要があります。（マイグレーションファイルをインポートさせないため、空の export をする必要がある。）

この時点で TypeScript のエラーが発生するが一旦無視してすすめる。

## build

一度 build を実行し、types を生成する。

```sh
npm run postinstall
```

`types/truffle-contracts/*.d.ts`が生成され、migrations-ts 上のエラーが消える。

## マイグレーションファイルのトランスパイル

truffle の migation コマンドは js ファイルしか読めないので、`migration-ts` 内の ts ファイルを js 形式に変更します。

```bash
$ npm run build-migration-files
```

上記のコマンドが実行されると、`migirations`という js ファイルが入ったディレクトリができるので、以降、`truffle migrate`でマイグレーションする場合はそちらのファイルを参照することになります。

migration-ts 内のファイルを更新した場合は都度上記のコマンドを実行し js を更新することになります。（実際は migration ファイルを更新する頻度は少ないですが）

マイグレーションを実行する場合は Ganache 等 Ethereum のクライアントを立ち上げ、マイグレーションコマンド `npm run migrate` を実行します。

## テストの準備

とりあえず、テストが動作することを確かめるためテストファイルを追加します。

```bash
$ touch test/migration.test.ts
```

下記のようにします

```ts
const MigrationsContract = artifacts.require('Migrations');

contract('Migrations', () => {
  it('has been deployed successfully', async () => {
    const migrations = await MigrationsContract.deployed();
    assert(migrations, 'contracut was not deployed');
  });
});
```

テストを実行し、動作することを確認します。

```sh
$ npm run test
```

## まとめ

typescript で truffle のテンプレート環境を作る部分を説明しました。

ファイルを更新するたびにコンパイルが必要だったりするので、なにかファイルを watch する機能をつけたほうが効率ではあります、時間があればそのあたりも詰めていきたですね。
