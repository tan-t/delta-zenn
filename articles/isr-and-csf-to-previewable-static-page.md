---
title: "自社メディアを ISR を使って爆速にした話"
emoji: "📈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react", "vercel"]
published: true
---

# メディアの紹介

[Beyond Magazine](https://www.beyondmag.jp/) というメディアを自社で展開しています。

紙の雑誌を長年やっていたメンバーで作っている web メディアで、グラフィックがいけてるので見てみてください。
![](https://storage.googleapis.com/zenn-user-upload/umo6g8r6dcoj4pz2poy8srgbq8xn)

技術的には、Shifter Headless をバックエンド（CMS）として、Next.js でメディア本体を描画しています。

もともとのアーキテクチャとしてはこんな感じでした。

![](https://storage.googleapis.com/zenn-user-upload/a252628fe6648f9b2fe2118a.png)

# 課題

- 速度（特に FCP）が遅いときだと数秒かかる
- Shifter が落ちるとサイト全体が落ちてしまう（しかも割と頻繁に落ちる）

という課題がありました。

# ISR の導入

## 導入の経緯

原因としては、リクエストの都度、Shifter Headless から同期的に記事データをフェッチしてから SSR して返すという仕様になっていたことが大きかったです。

Shifter Headless を導入した意図の一つですが、コメント機能などの動的なコンテンツが比較的少ないメディアという特性もあり、静的ページを出力するようなアーキテクチャに変更すると特に FCP は改善することが予想されました。

静的ページ出力の方針としては、ほぼ毎日記事をアップするというメディアの特性上、記事を更新するごとに全体のビルドを行う SSG 方式ではなく、リクエストをトリガーとしてキャッシュを更新する ISR 方式が向いていると判断し、ISR を導入することにしました。

## 実装

基本的には、`getServerSideProps`を`getStaticProps`に変更し、`revalidate`秒数を返り値に設定するだけで完了です。

```diff
- export const getServerSideProps: GetServerSideProps<Props> = async (
-   context
- ) => {
+ export const getStaticProps: GetStaticProps<Props> = async (context) => {
...

return {
-   props: ...
+   props: ...,
+  revalidate: 1440
}
```

アーキテクチャとしては以下のようになります。

![](https://storage.googleapis.com/zenn-user-upload/b7fc19cd6f6dbb2bcd372bac.png)

結果、FCP は遅いときで 5-6 秒かかっていたところ、現在では 1.1 秒（サーバー応答時間は 300ms 程度）に短縮できました。（他にもボトルネックはあるため、継続的に改善はしていく必要ありますが・・・）

直感的にもかなり速くなったと思います。

また、ISR は最後に成功したレンダリングのキャッシュを保持してくれるため、バックエンドとなる Shifter Headless が落ちていた場合でもサイトが落ちることがなくなったのも良かったです。

# リアルタイムプレビューの対応

## 課題

さて、Beyond Magazine は紙の雑誌出身の編集者と一緒に作っています。

そもそも、wordpress ベースのヘッドレス CMS である Shifter Headless を選定したのも、ブロックエディタを利用して編集者が自分でレイアウトを組んで入稿できるようにしたかったためです。

今回、ISR を導入することで本番環境の表示はかなり速くなりましたが、記事の編集者は常にリアルタイムで入稿中の記事の見栄えを確認したいというニーズを抱えています。

Beyond Magazine では、同じ Shifter を向いている二つの Next.js アプリケーションで、「本番環境」「プレビュー環境」の二つの環境を実現しています。

「プレビュー環境」は、「本番環境」と同じ Next.js アプリケーションがデプロイされていますが、環境変数により公開ステータスでない記事も表示できるように挙動を振り分けています。

以前は SSR だったため、本番環境もプレビュー環境も常に最新の記事データを表示できており、結果的に快適なプレビューが行えていたのですが、ISR にしたことで編集者がプレビューするページが常に一回前のリクエスト時に生成されたキャッシュになってしまうという課題が次は発生しました。

![](https://storage.googleapis.com/zenn-user-upload/60fd19a77c742410525d5701.png)

## クライアントサイドフェッチの導入

ところで、ISR を導入しているのにも関わらず快適なプレビューが行えるサイトがあるのをご存じでしょうか？

・・・そう、Zenn です。

> Zenn の記事ページは ISR ではなく SSR でしょうか？
> ISR だと思っていたのですが、記事の編集が即時で反映されたので気になりました。

> 遅くなりすみません。記事ページは ISR です。著者本人によるアクセスの場合にはクライアントでのマウント後に再フェッチしています。

[当該のコメント](https://zenn.dev/link/comments/81756c4773d7f9)

これが Beyond Magazine でも利用できると思われたので、

- 環境変数がプロダクションの場合、ISR オンリー
- 環境変数がプレビューの場合、ISR 後に再フェッチ

という戦略をとることにしました。

![](https://storage.googleapis.com/zenn-user-upload/106127c749ff263d9df74a59.png)

## 実装

さて、とはいうもののクライアントサイドから Shifter に直アクセスさせるのは、フロントエンドに Shifter の認証情報を露出させることになったりするのでなるべくやりたくありません。

また、ほぼ全ページに共通する仕様にしたかったため、なるべく共通的に・楽に実装する方法を考えました。

### 戦略

Next.js には[API Routes](https://nextjs.org/docs/api-routes/introduction)という、ページではなく json を返す API を手軽に作れる仕組みがあります。

また、ISR を導入した時点で、各ページコンポーネントは Next.js に認識させるために`getStaticProps`という名前の、ページレンダリングに必要な Props を取得する関数を export しています。

この二つを組み合わせて、

- `getStaticProps`を **import** して、
- `getStaticProps`の結果を返す API を API Routes で作成する
- それをクライアントサイドから fetch で取得し、Props をオーバーライドする

という作戦でいきました。

### getStaticProps()の結果を返す API を作る

まずは`getStaticProps`の結果を返す API を手軽に作れるようにします。

```ts
// utils/IsrUtils.ts
import { GetStaticProps, NextApiHandler } from "next";

/**
 * getStaticPropsの関数を、CSRで利用する用のAPIハンドラに変換する
 * @param base
 * @returns
 */
export const toCSRPropsApi =
  <X extends GetStaticProps>(base: X): NextApiHandler =>
  (req, res) =>
    base({ params: req.query })
      .then((props) => res.status(200).json(props))
      .catch((e) => res.status(500).json(e));
```

この`toCSRPropsApi`という関数に、`getStaticProps`を渡すと目当ての API を生成することができるというユーティリティです。

```ts
// pages/api/posts/[slug].ts など、ページと同階層のAPIファイルとして
import { toCSRPropsApi } from "../../../../utils/isrUtils";
import { getStaticProps } from "../../../posts/[slug]";

export default toCSRPropsApi(getStaticProps);
```

こういう感じで使います。  
これで API の量産体制が整いました。

### 作成した API を呼ぶ HOC を作成する

次は、呼び出し側を実装します。

今回はなるべくページコンポーネントの実装を変えずに、

- SSG 時には Next.js が`getStaticProps`で取得した Props を受け取る
- マウント後、API から取得した Props をレンダリングする

という仕様を満たしたいと思います。

ページコンポーネントは随所で props を参照するので、  
hooks などを使って新しく API から取得した props を別の変数に入れるなどをすると、  
props の参照箇所に手を入れる必要が出てくるかもしれません。

なので、今回はページコンポーネントの内部に全く手を入れる必要のない、HOC を使った実装を試みました。

```tsx
// components/isr/Previewable.tsx
import { GetStaticPropsResult } from "next";
import { useRouter } from "next/router";
import React, { ComponentProps, useEffect, useState } from "react";

/**
 * 環境変数でCSRに切り替えるHOC
 */
export const Previewable = (Children: React.FC<any>) => {
  return (serverSideProps: ComponentProps<typeof Children>) => {
    const csrProps = useCSRForPreview();
    const props =
      csrProps && "props" in csrProps ? csrProps.props : serverSideProps;
    return <Children {...props} />;
  };
};

const useCSRForPreview = <X extends React.FC>() => {
  const { route, query } = useRouter();
  const [result, setResult] = useState<
    GetStaticPropsResult<ComponentProps<X>> | undefined
  >(undefined);

  useEffect(() => {
    if (!process.env.NEXT_PUBLIC_IS_PREVIEW) {
      return;
    }
    fetch(`/api/${window.location.pathname}`)
      .then((res) => res.json())
      .then(setResult);
  }, [route, query]);

  return result;
};
```

この状態で、「`Previewable`」で対象のページを囲ってあげると、環境変数`NEXT_PUBLIC_IS_PREVIEW`が設定されている環境では、マウント後やナビゲーション時に表示中の URL に相当する API を呼び、props を更新して描画させるようにできるようになりました。

### HOC を使う

あとは簡単です。  
ページコンポーネントを`Previewable`で囲います。

```diff
- export default PostPage;
+ export default Previewable(PostPage);
```

以上で完成です！

# 最後に

今回は、ISR を導入して静的なページを高速化・安定化させただけでなく、編集者の UX も考えてプレビュー環境では常に最新の情報を表示できるよう、マウント後に再フェッチ・再描画する HOC やユーティリティを実装しました。

私の所属する Seven Rich Group の横断技術支援組織 では、このように実際にサービスを利用する人と二人三脚でサービス開発をしています。

Beyond Magazine だけではなく、自社事業や出資先の事業を中心に、toB SaaS や toC Web アプリなど多様な事業に超上流（構想段階の壁打ち）から入り、必要なものは自分たちで作るという働き方で参画しています。

事業にコミットできるという事業会社の働き方と、多様な事業に関われるという受託会社的な働き方のまさにいいとこどりができる組織になっているので、もし興味がある方はぜひ twitter で DM などお気軽にお願いします！

[note もチェックしてください！](https://note.com/sevenrich/n/naba285829c3e)
