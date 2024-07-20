+++
title="Stripe API with Next.js"
[taxonomies]
tags=["Next.js", "Stripe"]
+++

Stripe の API を Next.js で動かしてみた備忘録です。

## 準備

Stripe の Business アカウントを取得。

[https://dashboard.stripe.com/register](https://dashboard.stripe.com/register)

Next.js をインストールし、Stripe のライブラリをインストールする。

```bash
# node.js用ライブラリ
npm install --save stripe
# react及びフロントエンド側ライブラリ
npm install --save @stripe/react-stripe-js @stripe/stripe-js
```

# カスタムの支払いフローを実装する

Stripe 決済の基本となる支払い（決済）を実装する。今回、Stripe が用意している構築済みの Checkout ページは利用せず、自前（カスタム可能な）のコンポーネントで画面を作ります。

下準備として、 `.env.local`ファイルに以下ように Stripe のダッシュボード画面に表示されている API キーを登録します。公開されている API キーはフロントエンド側からも呼び出せる定数として、接頭辞に `NEXT_PUBLIC_` を付けています。

```
NEXT_PUBLIC_STRIPE_API_KEY=pk_test_...
STRIPE_SECRET_API_KEY=sk_test...
```

## PaymentIntent API を作成する

> PaymentIntent は、顧客の支払いライフサイクルを追跡し、支払いの試行失敗があった場合にはその記録を残して顧客への請求に重複が発生しないようにします。PaymentIntent の client secret をレスポンスで返して、クライアントで支払いを完了します。

実際の決済の前処理として、PaymentIntent を作成します。

Next.js の API Routes を利用し、チェックアウト用の API `pages/api/checkout.ts` を作成し、以下の内容を登録します。今回は純粋に `price` という金額(JPY)を受け取りそれを処理します。実際には購入アイテムのデータを受け取り、DB 等に問い合わせ正確な金額を取得するなどの処理が必要になります。

```ts
// pages/api/checkout.ts
import type { NextApiRequest, NextApiResponse } from 'next';
import Stripe from 'stripe';

const stripe_api_key = process.env.STRIPE_SECRET_API_KEY;
const stripe = new Stripe(stripe_api_key, {
  apiVersion: '2020-08-27'
});

export default async (req: NextApiRequest, res: NextApiResponse) => {
  const { price } = req.body;
  // 注文アイテムの個数と、通貨を指定し、PaymentIntent を作成
  const paymentIntent = await stripe.paymentIntents.create({
    amount: price,
    currency: 'jpy'
  });
  res.send({
    clientSecret: paymentIntent.client_secret
  });
};
```

## フロントエンド(React.js)の実装

決済のページは以下のようになります。

```ts
// pages/checkout.tsx
import { NextPage } from 'next';
import React from 'react';

import { Elements as StripeElements } from '@stripe/react-stripe-js';
import { loadStripe } from '@stripe/stripe-js';

import CheckoutForm from '../components/CheckoutForm';

interface CheckoutProps {}

// loadScriptでstripeのスクリプトを呼び出す
const promise = loadStripe(process.env.NEXT_PUBLIC_STRIPE_API_KEY);

/**
 * Checkout
 */
const Checkout: NextPage<CheckoutProps> = () => (
  <section>
    <div className="mt-10 mx-auto w-96">
      {/* striptのプロパティにloadStripを渡す。こうすることで子のコンポーネントでstripeのサービスが利用できるのようになる */}
      <StripeElements stripe={promise}>
        <CheckoutForm />
      </StripeElements>
    </div>
  </section>
);

export default Checkout;
```

loadStripe でブラウザで動作する Stripe の js ファイルを読み込みます。（処理としては<head>タグ内に `<script src="[https://js.stripe.com/v3](https://js.stripe.com/v3)"></script>` が読み込まれてる）

CheckoutForm にてカード番号の入力と決済の処理を行います。

```ts
import * as React from 'react';

import { CardElement, useElements, useStripe } from '@stripe/react-stripe-js';
import { StripeCardElementChangeEvent } from '@stripe/stripe-js';

interface CheckoutFormProps {}

const cardOptions = {
  hidePostalCode: true,
  style: {
    base: {
      color: '#1a1a1a',
      fontFamily: 'Arial, sans-serif',
      fontSmoothing: 'antialiased',
      lineHeight: '1.4',
      fontSize: '16px',
      '::placeholder': {
        color: '#999'
      }
    }
  }
};

/**
 * CheckoutForm
 */
const CheckoutForm: React.FC<CheckoutFormProps> = () => {
  const price = 1000;
  const [succeeded, setSucceeded] = React.useState(false);
  const [error, setError] = React.useState(null);
  const [processing, setProcessing] = React.useState(false);
  const [disabled, setDisabled] = React.useState(true);
  const [clientSecret, setClientSecret] = React.useState('');
  // react hookを利用したstripeへのアクセス
  const stripe = useStripe();
  const elements = useElements();

  React.useEffect(() => {
    // ページロード時にPaymentIntentを作成
    window
      .fetch('/api/checkout', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ price })
      })
      .then((res) => {
        return res.json();
      })
      .then((data) => {
        setClientSecret(data.clientSecret);
      });
  }, []);

  // 入力時のチェック
  const handleChange = async (event: StripeCardElementChangeEvent) => {
    setDisabled(event.empty);
    setError(event.error ? event.error.message : '');
  };

  const handleSubmit = async (ev: React.FormEvent) => {
    ev.preventDefault();
    setProcessing(true);
    const payload = await stripe.confirmCardPayment(clientSecret, {
      payment_method: {
        card: elements.getElement(CardElement)
      }
    });
    if (payload.error) {
      setError(`Payment failed ${payload.error.message}`);
      setProcessing(false);
    } else {
      setError(null);
      setProcessing(false);
      setSucceeded(true);
    }
  };

  return (
    <form id="payment-form" onSubmit={handleSubmit}>
      <div className="my-2">&yen;{price} の支払い</div>
      {/* 成功時の表示　*/}
      {succeeded ? (
        <p>
          <span>支払いが完了しました。</span>
          <a href={`https://dashboard.stripe.com/test/payments`}>Stripe dashboard.</a>
          <span>で確認しましょう。</span>
        </p>
      ) : (
        <div>
          {/* CartElementはiframeを利用したクレジットカード入力フォームを提供する */}
          <div className="border border-gray-300 p-4 rounded">
            <CardElement onChange={handleChange} options={cardOptions} />
          </div>
          <button
            className="rounded bg-blue-500 text-white px-4 py-2 mt-2"
            disabled={processing || disabled || succeeded}
            id="submit"
          >
            <span>{processing ? 'sending...' : 'Pay now'}</span>
          </button>
          {/* エラーの表示 */}
          {error && <div role="alert">{error}</div>}
        </div>
      )}
    </form>
  );
};
export default CheckoutForm;
```

ページ読み込み時に `POST /api/checkout` を呼び出し PaymentIntent を作成します。API から受け取った固有の clientSecret のキーを state に設定します。この値を Submit 時にセットすることでセキュリティなどのチェックを Stripe 側で行うことができるようになります。

Stripe.js が提供する`<CardElement>` コンポーネントにカード番号の入力を簡単にできます。iframe で実行されており、Secure3D にも対応することができます。

Submit 時に `stripe.confirmCardPayment` を呼び出し決済を実行します。エラーが発生しない倍、成功とし表示を切り替えます。エラーの場合はその内容を表示します。
