---
title: "JSXをただのテンプレートエンジンとして書いてみる"
emoji: "📈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react","typescript"]
published: true
---

# やったこと
- サーバーサイドで、LINE Flex Messageを生成する必要があった
- JSONでUIを表現するものだが、コンポーネントを作りたかった
- 引数が来て、分岐や繰り返しを含む制御を経て、最終的にはJSON文字列を出力したい
- tsconfigでJSXをコンパイルできるみたいな設定があるので、JSXで書いてみた

# この記事は
- Reactやpreactの実装を解説するものではないです。
- 勉強がてらやってみたというだけの記事です。

# 環境
- ts 4.2.3
- その他、[こちら](https://www.typescriptlang.org/play?target=99#code/BTCUF4D4AIG8FgBQ1oGMD2A7AzgF2gEoCmAhqvuHEiqgE6m5ECiANkQLZGa4Bc0APAA1oRAB6NMAE2zQAYgGF+AQUgAaFcFE9BqgA610u7DyWqAdBdQALAJYtJ9TH2JlcAOXSSiAbQC6oaCgqZBRoelwAV1pMQlJyM3psdBYANyJgWAszfUNsVWs7By4eAvtHAF9VUVAAbmpoSvrZWhIAc05uPmAcox5YUqLMAH5nOPdPH19yiBgEEJRwqJie7DMBxyGzADMWElwwM3YSXWBMCagXeMTktNOIlhZVM69QUDMAK3QbTGAAcl-avVGiFrqkiF0VjwSJgAJ6qaDPcFyCKYcg2LDQAA+0DwtG+rQCQTmoWgNi2wFwMN0RHQWwRE0C4Eov1x+IBxJJYSIkWi9K8dXmDXqC25Sz56RWgJC5SQMsQSEp1LkikEMEoIEh0GEADI4OtirFXB4vNMoKzMASBQqqURDeRjbbKObWlicbg8Ra-FbEBgcPgADI2PCBaDdAy9OA2RjsHjOvym2b1Ra8-iQep4Ei0XD1DkoFZmKMcIH1LiSer8AD0acQcqQvuDAGUSOxdGwQ2BAomQaKU4Hg4X2OBYN5fiRfqpfgAjX5TKuypB1rBJNhmFjoVrAJstthgUCy0BgIA)を参照のこと

# やりたいことの整理
- JSON文字列がアウトプットのテンプレートを管理したい
- 再利用したいので、最終的に文字列を返す関数をコンポーネントとして管理したい

# コンパイラの挙動の確認
```tsx
(()=> {
const React = {}

const List = (props: {item:string[]})=> {
  return <>
  start
  {
    props.item
  }
  end
  </>
}

const Sample = () => {
  return <List item={['a','b']}/>
}

console.log(Sample())
})()
```

上記のソースは、以下のようにトランスパイルされる。

```js
"use strict";
(() => {
    const React = {};
    const List = (props) => {
        return React.createElement(React.Fragment, null,
            "start",
            props.item,
            "end");
    };
    const Sample = () => {
        return React.createElement(List, { item: ['a', 'b'] });
    };
    console.log(Sample());
})();
```

ここからわかることは、
- `<なにか props="aa">{children}</なにか>` は、`React.createElement(なにか,props,...children)` の形式になる。

ということである。

というか、それぐらいのことは普通に[公式ドキュメントに書いてある](https://ja.reactjs.org/docs/jsx-in-depth.html)が、実際にそういう挙動であることを確認するのはいいことだ。

# React.createElementを実装する

## シグネチャ

ということで、`React.createElement` の関数は以下のようなシグネチャを持つことになるだろう。

```ts
type FC<X> = ((props:PropsWithChildren<X>) => string) | string;
type PropsWithChildren<P> = P & { children?: ReactNode }
type ReactNode = string | string[];
type CreateElement = <X extends FC<A>,A>(f:X,props:A,...children?: ReactNode) => string;
```

今回はなんちゃってテンプレートエンジンが欲しいのであって、得たいのは`string`の返り値なので関数コンポーネントの返り値も`string`になる。

なので当然`ReactNode`の型も`string`または`string[]`で問題ない。

## 実装

さて、`CreateElement`には「`string`を返す関数、または`string`自身」と、「そのProps」と、「`children`」が渡ってくる。

したがって、普通にそれらを解決すればOKである。

（途中でめんどくなって`any`使っちゃいましたが、nodeが`FC<A> | string`で、propsが`A | null`ということが言えるはずなのでいい感じに型を書くとさらにいいと思います）

```ts
const React = {
  createElement: <X extends FC<A>,A>(x:X,props:A,...children: ReactNode[]) => {
    return React.resolve({...props,children:children},x);
  },
  resolve: (props:any, node: Function | string) => {
    if(typeof node === 'string'){
      return node;
    }
    return node(props);
  }
}
```

こんな感じ。

# React.Fragmentを実装する

## 要件

JSXでコンポーネントを書いていくと、どこかで`React.Fragment`に到達する。

今回の例でいうと、「`List`」コンポーネントがそれにあたる。


```tsx
const List = (props: {item:string[]})=> {
  return <>
  start
  {
    props.item
  }
  end
  </>
}
```

```js
const List = (props) => {
    return React.createElement(React.Fragment, null,
        "start",
        props.item,
        "end");
};
```

## シグネチャ
当然だが、`React.FC` の仕様を踏襲するはずだ。

```ts
type Fragment = (props: PropsWithChildren<null>) => string;
```

今回の場合、`Fragment`はPropsを取らないため、純粋に渡ってきた`children`を左から右に並べて表示すればよい。

```ts
const React = {
  createElement: <X extends FC<A>,A>(x:X,props:A,...children: ReactNode[]) => {
    return React.resolve({...props,children:children},x);
  },
  Fragment: (props: {children?:ReactNode[]}) => {
    return props.children?.flat().map(node=>React.resolve(null,node)).join('');
  },
  resolve: (props:any, node: Function | string) => {
    if(typeof node === 'string'){
      return node;
    }
    return node(props);
  }
}

```

こんな感じ。

# 最終形

生成されたコードは以下のようになる。
なおソースは[こちら](https://www.typescriptlang.org/play?target=99#code/BTCUF4D4AIG8FgBQ1oGMD2A7AzgF2gEoCmAhqvuHEiqgE6m5ECiANkQLZGa4Bc0APAA1oRAB6NMAE2zQAYgGF+AQUgAaFcFE9BqgA610u7DyWqAdBdQALAJYtJ9TH2JlcAOXSSiAbQC6oaCgqZBRoelwAV1pMQlJyM3psdBYANyJgWAszfUNsVWs7By4eAvtHAF9VUVAAbmpoSvrZWhIAc05uPmAcox5YUqLMAH5nOPdPH19yiBgEEJRwqJie7DMBxyGzADMWElwwM3YSXWBMCagXeMTktNOIlhZVM69QUDMAK3QbTGAAcl-avVGiFrqkiF0VjwSJgAJ6qaDPcFyCKYcg2LDQAA+0DwtG+rQCQTmoWgNi2wFwMN0RHQWwRE0C4Eov1x+IBxJJYSIkWi9K8dXmDXqC25Sz56RWgJC5SQMsQSEp1LkikEMEoIEh0GEADI4OtirFXB4vNMoKzMASBQqqURDeRjbbKObWlicbg8Ra-FbEBgcPgADI2PCBaDdAy9OA2RjsHjOvym2b1Ra8-iQep4Ei0XD1DkoFZmKMcIH1LiSer8AD0acQcqQvuDAGUSOxdGwQ2BAomQaKU4Hg4X2OBYN5fiRfqpfgAjX5TKuypB1rBJNhmFjoVrAJstthgUCy0BgIA)

```js
"use strict";
(() => {
    const React = {
        createElement: (x, props, ...children) => {
            return React.resolve({ ...props, children: children }, x);
        },
        Fragment: (props) => {
            return props.children?.flat().map(node => React.resolve(null, node)).join('');
        },
        resolve: (props, node) => {
            if (typeof node === 'string') {
                return node;
            }
            return node(props);
        }
    };
    const List = (props) => {
        return React.createElement(React.Fragment, null,
            "start",
            props.item,
            "end");
    };
    const Sample = () => {
        return React.createElement(List, { item: ['a', 'b'] });
    };
    console.log(Sample());
})();
```

実行すると（どこでも実行できる）、コンソールには  
```shell
startabend
```
と表示される。まあ、普通ですね・・・

# まとめ

結果的に勉強になったのでよかった。（感想）

JSXは通常のテンプレートエンジンに比べて、  
- 複雑な独自制御構文を覚える必要が無い。
- コンポーネントに切り出しやすく、かつコンポーネントが型安全に書ける。
- tsとの親和性抜群。

というメリットがあるため、サーバーサイドでのちょっとしたテンプレート利用でも活用していいのでは？と思いました。







