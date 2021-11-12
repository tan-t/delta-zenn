---
title: "2021年版 lambdaでPDF作成（日本語対応）"
emoji: "📜"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["lambda", "puppeteer", "serverless"]
published: false
---

<font size="64">みなさま～～～！</font>

<font size="64">lambda で PDF、つくりたいですよね？</font>

# lambda で PDF をつくるのは大変

つくりたいのはやまやまだとしても・・・

- HTML から PDF をつくる node.js ライブラリはいろいろあるが、だいたいブラウザを必要とする。
- lambda のランタイムにはブラウザが入っていないため、lambda 上で使おうとすると大変。

などなど、大変なことがいっぱいあるので断念した方も多いのではないでしょうか。

# 今回の前提

- serverless framework + webpack で既存アプリを構築している上に、
- lambda の node.js ランタイムで、
- 一番実装が速い方法で、
- 日本語対応したうえで、
- HTML を PDF にする機能を**追加**する。

# 避けたいこと

- パッケージの容量を(必要以上に)大きくしたくない
- 開発者のディレクトリに不要なものを置きたくない（バイナリ、フォントファイルなど）
- lambda ランタイムの OS やディレクトリ構成を意識したくない

# 考慮しないこと

- ローカル開発ビリティ

  - ローカルで開発する場合は開発環境にインストールされているブラウザを直接操作するか docker 等でヘッドレスブラウザがインストールされている環境を固定すればよいと思われますが、本稿では割愛します。

# やること

## 戦略

[chrome-aws-lambda](https://github.com/alixaxel/chrome-aws-lambda) を利用します。  
このパッケージは、lambda のコンテナに含まれていないブラウザのバイナリを含んでおり、このパッケージを lambda にデプロイすると lambda 上で puppeteer を動かせるようになるという代物です。

ただ、普通に利用しようとするとパッケージにバイナリを入れる必要があるのでパッケージが大きくなってしまったり、lambda layers を自分でデプロイするなど面倒な手順が多いので、  
今回は[既に各リージョンで配布されている lambda layers](https://github.com/shelfio/chrome-aws-lambda-layer#available-regions)を利用します。

## 実装

### 関数を追加

```diff
# serverless.yml
---
functions:
+  pdf:
+    handler: src/handler.pdf
+    layers:
+      - arn:aws:lambda:ap-northeast-1:764866452798:layer:chrome-aws-lambda:22
---
```

まず、serverless.yml に対象の layers を持つ関数を追加します。  
関数名等はおこのみで。  
リージョンはいったん東京としています。

### webpack.config.js に externals を追加

webpack 環境の場合、

- chrome-aws-lambda は layers に既に含まれている
- chrome-aws-lambda を webpack のパッケージ対象に入れる必要が無い

ため、webpack のパッケージ対象から外します。

```diff
// webpack.config.js

module.exports = {
  // ...
+  externals: ['chrome-aws-lambda'],
  // ...
}

```

### 実装

#### puppeteer を呼び出す

chrome-aws-lambda は、puppeteer を内包しているので out of the box で利用できます。  
日本語フォントが含まれていないため、CDN 経由で noto sans を読み込んだうえで puppeteer のブラウザインスタンスを返すユーティリティをまず作っておきましょう。

```ts
const getPuppeteer = async () => {
  // TODO 開発環境のOSのブラウザを利用するpuppeteerを返却する
  //   if (process.env.IS_OFFLINE) {
  //   }

  const chromium = require("chrome-aws-lambda");

  // CDNから日本語フォントを読み込む
  await chromium.font(
    "https://raw.githack.com/minoryorg/Noto-Sans-CJK-JP/master/fonts/NotoSansCJKjp-Regular.ttf"
  );

  return chromium.puppeteer.launch({
    executablePath: await chromium.executablePath,
    args: chromium.args,
    headless: chromium.headless,
    defaultViewport: chromium.defaultViewport,
  });
};
```

記事によっては`.fonts`ディレクトリにフォントファイルを配置してデプロイパッケージに含めるという方法をとる実装が紹介されていることが多いですが、今回は開発環境に余計なバイナリを置きたくない（`.fonts`ディレクトリをリポジトリにコミットするのが気持ち悪い）のでインターネットから読み込む実装をしています。

#### 関数本体を実装

puppeteer のブラウザインスタンスが作成されさえすれば普通に実装が可能です。
以下は https://google.co.jp にアクセスして A4 の PDF にして返す API Gateway トリガーの関数の例です。

```ts
// src/handler.ts

export const pdf = async (event, context) => {
  const browser = await getPuppeteer();
  const page = await browser.newPage();
  // A4
  await page.setViewport({
    width: Number(2480),
    height: Number(3508),
  });
  // ページ遷移
  await page.goto("https://google.co.jp");
  const pdf = await page.pdf();
  return {
    status: 200,
    body: pdf.toString("base64"),
    headers: {
      "Content-Type": "application/pdf",
    },
  };
};
```

## 以上です

これだけです。

# TODO

今回は puppeteer のブラウザインスタンスを返す関数のローカル開発時の挙動の実装を省きました。

ここの実装は各自やってみてください。

# we're hiring!

今回は、serverless framework 環境下で、パッケージを重くすることなく、素早く PDF 描画機能を実装しました。

私の所属する Seven Rich Group の横断技術支援組織 では、サービスを利用する人と二人三脚でサービス開発をしています。

自社事業や出資先の事業を中心に、toB SaaS や toC Web アプリなど多様な事業に超上流（構想段階の壁打ち）から入り、必要なものは自分たちで作るという働き方で参画しています。

事業にコミットできるという事業会社の働き方と、多様な事業に関われるという受託会社的な働き方のまさにいいとこどりができる組織になっているので、もし興味がある方はぜひ twitter で DM などお気軽にお願いします！

[note もチェックしてください！](https://note.com/sevenrich/n/naba285829c3e)

# 参考記事

- [[chrome-aws-lambda]文字化けする時の簡単な解決方法
  ](https://yongjinkim.com/chrome-aws-lambda%E6%96%87%E5%AD%97%E5%8C%96%E3%81%91%E3%81%99%E3%82%8B%E6%99%82%E3%81%AE%E7%B0%A1%E5%8D%98%E3%81%AA%E8%A7%A3%E6%B1%BA%E6%96%B9%E6%B3%95/)
